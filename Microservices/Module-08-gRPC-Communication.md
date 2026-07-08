# Module 8 — gRPC Communication

> **Microservices Masterclass** | Level: Intermediate | Track: Node.js Backend Engineering
> Prerequisite: Module 1–7 (especially Module 7 — REST Communication)
> Next Module: Module 9 — Event Driven Architecture

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

Module 7 gave you REST — flexible, universally understood, and perfect for public-facing or loosely-coupled APIs. But REST has real costs: JSON parsing overhead, no strict contract enforcement (a typo in a field name fails silently at runtime, not at compile/build time), and HTTP/1.1's inefficiency for high-volume internal traffic.

**gRPC** is Google's answer to a very specific problem: **high-performance, strongly-typed communication between services you control**, where you want a strict contract enforced automatically, minimal serialization overhead, and native support for streaming data — none of which REST does particularly well out of the box.

This module isn't about replacing REST — it's about adding a second tool to your toolbox and knowing precisely when to reach for it. By the end, you'll be able to build a working gRPC service in Node.js and make a confident, well-reasoned choice between REST and gRPC for any given internal integration.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain what gRPC is and how it differs fundamentally from REST.
- Write and use Protocol Buffer (`.proto`) definitions to define a strict service contract.
- Implement a gRPC server and client in Node.js.
- Explain and choose between gRPC's four call types: unary, server streaming, client streaming, and bidirectional streaming.
- Identify concrete scenarios where gRPC is the better choice over REST, and where REST remains preferable.
- Understand gRPC's performance characteristics and how HTTP/2 enables them.

---

## 3. Problem Statement

A high-throughput internal service, `pricing-service`, is called synchronously by `order-service` on every single item in every cart, potentially thousands of times per second during peak traffic. With REST:

- Every call incurs JSON serialization/deserialization overhead — noticeable at this volume.
- A typo like `unitPrice` vs `unit_price` in a hand-maintained API contract isn't caught until runtime, potentially in production.
- HTTP/1.1 (the default for most REST setups) opens a new connection or serializes requests over limited connections per host, adding latency under very high concurrency, since only newer setups leverage HTTP/2 for REST.
- The team wants to stream a continuous feed of live price updates to `order-service` rather than polling repeatedly — REST has no clean, native way to do this (you'd need to bolt on WebSockets or Server-Sent Events separately).

This module introduces gRPC as the solution built specifically for this class of problem: internal, high-throughput, strongly-typed, and potentially streaming communication between services your own organization controls.

---

## 4. Why This Concept Exists

gRPC exists because REST + JSON, while excellent for flexibility and universal compatibility, makes trade-offs that don't fit every internal communication scenario:

| REST + JSON Trade-off | gRPC's Answer |
|---|---|
| JSON is human-readable but verbose and slower to parse | Protocol Buffers: compact binary format, much faster to serialize/deserialize |
| No enforced contract — mismatched fields fail at runtime | `.proto` files define a strict, code-generated contract — many errors caught at compile/build time |
| HTTP/1.1 has per-connection request limits | Built on HTTP/2: multiplexing many requests over a single connection, header compression |
| No native streaming — must bolt on WebSockets/SSE | Native support for streaming (server, client, and bidirectional) as a core protocol feature |
| Great for public APIs (wide client compatibility) | Best for internal service-to-service calls where both ends are under your control and can share generated code |

gRPC deliberately sacrifices some of REST's universal simplicity (not every client/browser can easily consume gRPC without extra tooling) in exchange for performance and contract safety — making it the right tool specifically for **internal, high-performance, controlled-environment** communication.

---

## 5. Historical Background

- **2001** — Google began developing an internal RPC framework called "Stubby," used extensively within Google's own infrastructure for years, to handle the massive volume of internal service-to-service calls at Google's scale.
- **2015** — Google open-sourced a public version of this technology as **gRPC** ("gRPC Remote Procedure Calls"), built on **HTTP/2** (standardized in 2015) and **Protocol Buffers** (Google's binary serialization format, first released internally in 2001 and open-sourced in 2008).
- **2015 onward** — gRPC was donated to the **Cloud Native Computing Foundation (CNCF)**, the same foundation that hosts Kubernetes, reflecting its adoption as a standard building block for cloud-native, microservices-based systems.
- **Present** — gRPC is widely used for internal service-to-service communication at companies operating at significant scale (Google, Netflix, Square, and many others), particularly for latency-sensitive or very high-throughput internal calls, while REST remains dominant for public-facing and less performance-critical APIs.

> **Interview tip:** A common and important interview point: gRPC is generally **not** used for public-facing browser-facing APIs directly (browsers don't natively support raw HTTP/2 trailers the way gRPC needs, though gRPC-Web exists as a workaround) — it shines specifically for internal, service-to-service communication.

---

## 6. Real-World Analogy

**Analogy: A Standardized Shipping Manifest vs. a Handwritten Note**

**REST + JSON is like sending a package with a handwritten note describing its contents.** It's flexible and anyone can read it, but the receiving warehouse has to carefully parse potentially inconsistent handwriting, and there's no guarantee the sender used the exact same terms you expect ("qty" vs "quantity" vs "count").

**gRPC + Protocol Buffers is like using a standardized, pre-printed shipping manifest form with fixed fields, agreed upon in advance by both the sender and receiver's systems, and encoded compactly (like a barcode) rather than handwritten prose.** Both sides' systems already know exactly what fields exist, in what order, and what type each one is — because they were both generated from the same official manifest template (`.proto` file). This is faster to process (a barcode scanner beats reading handwriting) and virtually eliminates a whole category of miscommunication errors, but it does require both sides to have agreed on and adopted that specific standardized form ahead of time — you can't just wing it with a new format on the fly, the way you could with a handwritten note.

---

## 7. Technical Definition

> **gRPC** is a high-performance, open-source Remote Procedure Call (RPC) framework, built on **HTTP/2**, that uses **Protocol Buffers (protobuf)** as its default interface definition language and binary serialization format, enabling strongly-typed, contract-first service-to-service communication with native support for streaming.

> A **`.proto` file** defines a service's contract: its available RPC methods, and the structure (message types) of their requests and responses — this file is the **single source of truth**, from which client and server code is automatically generated in multiple languages.

> gRPC supports four call patterns:
> - **Unary RPC**: one request, one response (analogous to a typical REST request-response call).
> - **Server Streaming RPC**: one request, a stream of multiple responses over time.
> - **Client Streaming RPC**: a stream of multiple requests, one final response.
> - **Bidirectional Streaming RPC**: both sides send a stream of messages independently, over the same long-lived connection.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Protocol Buffers (protobuf)** | Google's language-neutral, binary serialization format and interface definition language |
| **`.proto` file** | The contract definition file: services, RPC methods, and message schemas |
| **Stub** | The auto-generated client-side code that lets you call remote gRPC methods as if they were local functions |
| **Unary RPC** | One request → one response (most similar to typical REST calls) |
| **Server Streaming** | One request → a stream of responses over time |
| **Client Streaming** | A stream of requests → one final response |
| **Bidirectional Streaming** | Both sides stream messages independently over one connection |
| **HTTP/2** | The underlying transport protocol gRPC uses, supporting multiplexing multiple requests over a single TCP connection |
| **Channel** | A persistent connection abstraction a gRPC client uses to communicate with a server |
| **Interceptor** | gRPC's equivalent of middleware — used for logging, auth, metrics on both client and server sides |
| **Deadline** | gRPC's built-in, first-class concept of a request timeout, propagated automatically through a call chain |

---

## 9. Internal Working

Here's how a gRPC call actually works end-to-end:

1. Both the client and server are built from the **same `.proto` file**, using the `protoc` compiler (or a JS equivalent) to generate strongly-typed client stubs and server interfaces in their respective languages.
2. The client calls a method on its generated **stub** as if it were a local function (e.g., `client.getPrice(request)`), even though this transparently triggers a network call.
3. The request message is serialized into **Protocol Buffers' compact binary format** — significantly smaller and faster to parse than equivalent JSON.
4. The serialized message is sent over an **HTTP/2** connection, which supports multiplexing many concurrent requests/streams over a single TCP connection, reducing connection overhead compared to typical HTTP/1.1 usage.
5. The server's generated code deserializes the binary message back into a strongly-typed object matching the `.proto` schema, and routes it to your actual business logic implementation.
6. The server's response (also a `.proto`-defined message) is serialized and sent back the same way.
7. For streaming calls, instead of a single request/response, either side (or both) can send a continuous sequence of messages over the same long-lived connection, without needing to establish a new connection per message.
8. Deadlines (timeouts) are propagated automatically through the call — if Service A calls B with a 2-second deadline, and B calls C, that deadline context can be passed along so C also knows the overall time budget remaining.

---

## 10. Step-by-Step Request Flow

**Scenario: `order-service` calls `pricing-service` via gRPC to get live prices for cart items (unary call).**

```
Step 1:  Both order-service and pricing-service are built from the
         SAME pricing.proto file, generating matching client/server code
Step 2:  order-service constructs a GetPriceRequest message
         { productId: "abc123" }
Step 3:  order-service calls the generated stub method: client.getPrice(request)
Step 4:  The request is serialized into Protocol Buffers binary format
Step 5:  Sent over an HTTP/2 connection to pricing-service (multiplexed
         alongside any other concurrent calls on the same connection)
Step 6:  pricing-service's generated server code deserializes the
         binary message into a strongly-typed GetPriceRequest object
Step 7:  pricing-service's business logic looks up the price
Step 8:  pricing-service constructs and returns a GetPriceResponse message
         { productId: "abc123", price: 29.99, currency: "USD" }
Step 9:  The response is serialized, sent back over HTTP/2
Step 10: order-service's stub deserializes it into a strongly-typed
         object and returns it to the calling code, as if it were
         a normal local async function call
```

---

## 11. Architecture Overview

```
                  order-service
                        │
              ┌─────────┴─────────┐
              │  Generated gRPC     │
              │  Client Stub        │
              │  (from pricing.proto)│
              └─────────┬─────────┘
                        │
                  HTTP/2 Channel
              (multiplexed, persistent)
                        │
              ┌─────────┴─────────┐
              │  Generated gRPC     │
              │  Server Interface    │
              │  (from pricing.proto)│
              └─────────┬─────────┘
                        │
                  pricing-service
                  (business logic)
                        │
                        ▼
                   Pricing DB
```

---

## 12. ASCII Diagrams

### 12.1 gRPC's Four Call Types

```
UNARY (one request, one response):

  Client ──request──▶ Server
  Client ◀─response── Server


SERVER STREAMING (one request, many responses):

  Client ──request──▶ Server
  Client ◀────response 1─── Server
  Client ◀────response 2─── Server
  Client ◀────response 3─── Server (stream ends)


CLIENT STREAMING (many requests, one response):

  Client ──request 1──▶ Server
  Client ──request 2──▶ Server
  Client ──request 3──▶ Server
  Client ◀──final response── Server


BIDIRECTIONAL STREAMING (both sides stream independently):

  Client ──msg A──▶ Server
  Client ◀──msg X── Server
  Client ──msg B──▶ Server
  Client ◀──msg Y── Server
  (both continue independently over the same connection)
```

### 12.2 REST vs gRPC Wire Format Comparison

```
REST + JSON (text-based, larger payload):

  {"productId":"abc123","quantity":2,"unitPrice":29.99}
  ~55 bytes, requires text parsing


gRPC + Protocol Buffers (binary, compact):

  [binary encoded bytes representing the same fields,
   using field numbers instead of field names]
  ~15-20 bytes, no text parsing overhead, faster to
  serialize/deserialize at high volume
```

### 12.3 HTTP/1.1 vs HTTP/2 Connection Usage

```
HTTP/1.1 (typical REST usage):

  Client ──Connection 1──▶ Request A (waits for response before next on this connection)
  Client ──Connection 2──▶ Request B
  Client ──Connection 3──▶ Request C
  (often limited concurrent connections per host)


HTTP/2 (gRPC's foundation):

  Client ══Single Connection══▶ Request A, B, C ALL multiplexed
                                 concurrently over ONE connection
```

---

## 13. Mermaid Flowcharts

### 13.1 REST vs gRPC Decision Flow

```mermaid
flowchart TD
    A[New internal service integration] --> B{Is this a public-facing or browser-facing API?}
    B -- Yes --> C[Use REST]
    B -- No, internal only --> D{Do you need streaming - live updates, large data feeds?}
    D -- Yes --> E[Use gRPC - streaming support]
    D -- No --> F{Is this extremely high-throughput / latency-sensitive?}
    F -- Yes --> G[Use gRPC - performance + strict contract]
    F -- No --> H[REST is simpler and likely sufficient]
```

### 13.2 gRPC Call Flow

```mermaid
flowchart LR
    A[.proto file] --> B[protoc code generation]
    B --> C[Client Stub - JS]
    B --> D[Server Interface - JS]
    C --> E[Client calls stub method]
    E --> F[Serialize to Protobuf binary]
    F --> G[Send over HTTP/2]
    G --> H[Server deserializes]
    H --> D
    D --> I[Business logic executes]
    I --> J[Response serialized, sent back]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Unary gRPC Call

```mermaid
sequenceDiagram
    participant O as Order Service (client)
    participant P as Pricing Service (server)

    O->>P: getPrice(GetPriceRequest{productId: "abc123"}) [gRPC/HTTP2]
    P->>P: Look up price in database
    P-->>O: GetPriceResponse{price: 29.99, currency: "USD"}
```

### 14.2 Server Streaming gRPC Call (live price updates)

```mermaid
sequenceDiagram
    participant O as Order Service (client)
    participant P as Pricing Service (server)

    O->>P: subscribeToPriceUpdates(SubscribeRequest{productId: "abc123"})
    P-->>O: PriceUpdate{price: 29.99} (t=0s)
    P-->>O: PriceUpdate{price: 27.99} (t=30s, price dropped)
    P-->>O: PriceUpdate{price: 31.99} (t=60s, price rose)
    Note over O,P: Single long-lived connection, server pushes<br/>updates whenever the price changes - no polling needed
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                    Pricing Service (gRPC Server)             │
│  ┌───────────────────┐                                       │
│  │  pricing.proto       │  <- single source of truth           │
│  │  (service contract)   │                                     │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌───────────────────┐                                       │
│  │  Generated Server     │  <- auto-generated from .proto        │
│  │  Interface (JS)        │                                     │
│  └─────────┬───────────┘                                     │
│            ▼                                                  │
│  ┌───────────────────┐        ┌──────────────────┐           │
│  │  Business Logic       │──────▶│   Pricing DB       │           │
│  │  Implementation       │        └──────────────────┘           │
│  └───────────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                               │
│  order-svc pods ──gRPC/HTTP2 (via ClusterIP Service)──▶ pricing-svc pods │
│                                                               │
│  Note: Kubernetes Services natively support gRPC traffic,     │
│  but load balancing gRPC's long-lived HTTP/2 connections       │
│  requires care — simple L4 (TCP) load balancing can send ALL   │
│  requests on one persistent connection to the SAME pod,        │
│  causing uneven load. Solutions: client-side load balancing,   │
│  a gRPC-aware L7 proxy/service mesh (e.g., Envoy, Istio,        │
│  Linkerd), or periodically cycling connections.                │
└───────────────────────────────────────────────────────────┘
```

---

## 17. Database Interaction

gRPC itself is a communication protocol and has no inherent opinion on database interaction — `pricing-service` still owns its own database exclusively, exactly as in previous modules:

```
pricing-service (gRPC server)
       │
       ▼
  Pricing DB (owned exclusively by pricing-service)

order-service NEVER queries Pricing DB directly —
it only calls pricing-service's gRPC interface,
exactly as it would call a REST API — the communication
PROTOCOL changed (REST -> gRPC), but the fundamental
"database per service" rule from Module 3 is unchanged.
```

---

## 18. Failure Scenarios

| Scenario | gRPC Behavior |
|---|---|
| Server unreachable | Client receives a gRPC status code (`UNAVAILABLE`) — analogous to a REST 503, handled similarly with retries/circuit breakers |
| Deadline exceeded | Client receives `DEADLINE_EXCEEDED` — gRPC's built-in equivalent of a timeout, but propagated natively through the call chain |
| Invalid request (schema mismatch) | Often caught at compile/build time due to the strict `.proto` contract, reducing a whole class of runtime failures common in loosely-typed REST/JSON APIs |
| Server-side error during business logic | Returns a specific gRPC status code (e.g., `INVALID_ARGUMENT`, `NOT_FOUND`, `INTERNAL`) with an optional error message/details |
| Streaming call interrupted mid-stream | Client/server must handle partial streams gracefully — e.g., resuming or restarting the stream, since gRPC doesn't automatically replay missed messages |

```
gRPC Status Codes (a small selection, analogous to HTTP status codes):

  OK                  -> success (like 200)
  INVALID_ARGUMENT     -> like 400
  NOT_FOUND            -> like 404
  DEADLINE_EXCEEDED    -> like a timeout
  UNAVAILABLE          -> like 503
  INTERNAL             -> like 500
```

---

## 19. Scalability Discussion

gRPC's use of HTTP/2 multiplexing allows many concurrent calls over a single connection, reducing the connection-management overhead that can become a bottleneck for very high-throughput REST/HTTP1.1 setups. Its compact binary serialization also reduces CPU spent on parsing at high request volumes compared to JSON. However, gRPC's long-lived HTTP/2 connections introduce a **load balancing nuance**: naive Layer 4 (TCP) load balancers may pin a client to a single server connection, causing uneven load distribution across server replicas — this requires either client-side load balancing (the client itself is aware of multiple server addresses and distributes calls) or an L7-aware proxy/service mesh capable of properly load balancing individual gRPC requests within a shared connection.

---

## 20. High Availability Considerations

- As with REST, run multiple replicas of any gRPC server behind appropriate load balancing (with the nuance noted in Section 19).
- gRPC's built-in **deadline propagation** helps prevent one slow downstream call in a chain from silently consuming the entire time budget of the original caller's request — improving overall system responsiveness under partial failure.
- Client-side retry policies can be configured declaratively in some gRPC client libraries (with backoff and retryable status code lists), similar in spirit to the manual retry logic built in Module 7 for REST.

---

## 21. CAP Theorem Implications

Like REST, gRPC unary calls are synchronous and favor **Consistency** — the caller waits for a definitive answer. gRPC's streaming modes don't fundamentally change this trade-off; they simply allow the "response" to be an ongoing sequence rather than a single message, still within a live, synchronous-style connection. If you need to favor Availability during a partition (accepting eventual consistency), you'd still reach for asynchronous, event-driven communication (Module 9's Kafka-based patterns) rather than gRPC, regardless of which streaming mode you use.

---

## 22. Node.js Implementation

Let's build a working gRPC server and client for the `pricing-service` example.

**Folder structure:**
```
pricing-service/
├── protos/
│   └── pricing.proto
├── src/
│   └── server.js
└── package.json

order-service/
├── protos/
│   └── pricing.proto   (shared copy of the same contract)
├── src/
│   └── pricingClient.js
```

**`protos/pricing.proto`** (the shared contract — single source of truth)
```protobuf
syntax = "proto3";

package pricing;

// The service definition: what RPC methods are available
service PricingService {
  // Unary RPC: one request, one response
  rpc GetPrice (GetPriceRequest) returns (GetPriceResponse);

  // Server streaming RPC: one request, a stream of responses
  rpc SubscribeToPriceUpdates (SubscribeRequest) returns (stream PriceUpdate);
}

message GetPriceRequest {
  string product_id = 1;
}

message GetPriceResponse {
  string product_id = 1;
  double price = 2;
  string currency = 3;
}

message SubscribeRequest {
  string product_id = 1;
}

message PriceUpdate {
  string product_id = 1;
  double price = 2;
  int64 updated_at = 3;
}
```

**`pricing-service/src/server.js`**
```javascript
import grpc from "@grpc/grpc-js";
import protoLoader from "@grpc/proto-loader";
import { getPriceFromDb, subscribeToPriceChanges } from "./priceRepository.js";

const packageDefinition = protoLoader.loadSync("protos/pricing.proto", {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});
const pricingProto = grpc.loadPackageDefinition(packageDefinition).pricing;

// Unary RPC implementation
async function getPrice(call, callback) {
  try {
    const { product_id } = call.request;
    const priceData = await getPriceFromDb(product_id);
    if (!priceData) {
      return callback({
        code: grpc.status.NOT_FOUND,
        message: `Product ${product_id} not found`,
      });
    }
    callback(null, {
      product_id,
      price: priceData.price,
      currency: priceData.currency,
    });
  } catch (err) {
    callback({ code: grpc.status.INTERNAL, message: err.message });
  }
}

// Server streaming RPC implementation
function subscribeToPriceUpdates(call) {
  const { product_id } = call.request;

  // Push a new message on the stream every time the price changes
  const unsubscribe = subscribeToPriceChanges(product_id, (updatedPrice) => {
    call.write({
      product_id,
      price: updatedPrice,
      updated_at: Date.now(),
    });
  });

  // Clean up when the client disconnects
  call.on("cancelled", () => unsubscribe());
}

const server = new grpc.Server();
server.addService(pricingProto.PricingService.service, {
  getPrice,
  subscribeToPriceUpdates,
});

server.bindAsync(
  "0.0.0.0:50051",
  grpc.ServerCredentials.createInsecure(), // use TLS credentials in production
  () => {
    console.log("Pricing gRPC service running on port 50051");
    server.start();
  }
);
```

---

## 23. Express.js Examples

While gRPC servers don't use Express (they use the gRPC server framework directly, as shown above), it's common for a service to expose **both** a gRPC interface (for internal, high-performance calls) and a REST/Express interface (for external or lower-volume needs) side by side. Here's the gRPC **client** being used from within an Express-based `order-service`:

```javascript
// order-service/src/pricingClient.js
import grpc from "@grpc/grpc-js";
import protoLoader from "@grpc/proto-loader";

const packageDefinition = protoLoader.loadSync("protos/pricing.proto", {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});
const pricingProto = grpc.loadPackageDefinition(packageDefinition).pricing;

const client = new pricingProto.PricingService(
  process.env.PRICING_SERVICE_GRPC_URL || "pricing-service:50051",
  grpc.credentials.createInsecure() // use TLS credentials in production
);

// Wrap the callback-based gRPC client in a Promise for easy async/await use
export function getPrice(productId) {
  return new Promise((resolve, reject) => {
    const deadline = new Date(Date.now() + 3000); // 3-second deadline
    client.getPrice({ product_id: productId }, { deadline }, (err, response) => {
      if (err) return reject(err);
      resolve(response);
    });
  });
}
```

```javascript
// order-service/src/app.js
import express from "express";
import { getPrice } from "./pricingClient.js";

const app = express();
app.use(express.json());

app.get("/orders/preview-price/:productId", async (req, res) => {
  try {
    const price = await getPrice(req.params.productId);
    res.json(price);
  } catch (err) {
    if (err.code === grpc.status.NOT_FOUND) {
      return res.status(404).json({ error: "Product not found" });
    }
    res.status(502).json({ error: "Pricing service unavailable" });
  }
});

app.listen(4002, () => console.log("Order Service running on port 4002"));
```

This shows the practical pattern: `order-service`'s REST API (for its own external/client-facing needs) internally calls `pricing-service` via gRPC (for the internal, high-performance leg of the request) — the two protocols coexist naturally in one system.

---

## 24. Docker Examples

```yaml
version: "3.9"
services:
  pricing-service:
    build: ./pricing-service
    ports:
      - "50051:50051"  # gRPC port
    environment:
      - DATABASE_URL=postgresql://user:pass@pricing-db:5432/pricing
    depends_on: [pricing-db]

  order-service:
    build: ./order-service
    ports:
      - "4002:4002"    # REST port (for its own clients)
    environment:
      - PRICING_SERVICE_GRPC_URL=pricing-service:50051
    depends_on: [pricing-service]

  pricing-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=pricing]
```

```dockerfile
# pricing-service/Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 50051
USER node
CMD ["node", "src/server.js"]
```

---

## 25. Kafka/Redis Integration

gRPC (synchronous) and Kafka (asynchronous) serve different purposes and commonly coexist in the same service, exactly as REST and Kafka did in earlier modules:

```javascript
// pricing-service: after a price change is persisted, publish an
// ASYNCHRONOUS event for services that don't need the gRPC streaming
// interface (e.g., an Analytics service just logging historical prices),
// WHILE ALSO pushing the change over the gRPC server-streaming
// connection to any actively subscribed clients (as in Section 22).
export async function updatePrice(productId, newPrice) {
  await db.query("UPDATE prices SET price = $1 WHERE product_id = $2", [newPrice, productId]);

  // Push to any live gRPC stream subscribers
  notifyStreamSubscribers(productId, newPrice);

  // Also publish an async event for other, decoupled consumers
  await kafkaProducer.send({
    topic: "price-events",
    messages: [{ key: productId, value: JSON.stringify({ productId, newPrice }) }],
  });
}
```

Redis can be used as a fast local cache in front of gRPC calls exactly as it was for REST calls in Module 7 — the caching pattern is protocol-agnostic.

---

## 26. Error Handling

gRPC has a well-defined, standardized set of **status codes** (distinct from, but conceptually similar to, HTTP status codes) that should be used deliberately:

```javascript
import grpc from "@grpc/grpc-js";

async function getPrice(call, callback) {
  const { product_id } = call.request;

  if (!product_id) {
    return callback({
      code: grpc.status.INVALID_ARGUMENT,
      message: "product_id is required",
    });
  }

  const priceData = await getPriceFromDb(product_id);
  if (!priceData) {
    return callback({
      code: grpc.status.NOT_FOUND,
      message: `Product ${product_id} not found`,
    });
  }

  callback(null, { product_id, price: priceData.price, currency: priceData.currency });
}
```

On the client side, handle each relevant status code explicitly:
```javascript
client.getPrice({ product_id: "xyz" }, (err, response) => {
  if (err) {
    switch (err.code) {
      case grpc.status.NOT_FOUND:
        // handle missing product
        break;
      case grpc.status.DEADLINE_EXCEEDED:
        // handle timeout — consider retry if idempotent
        break;
      case grpc.status.UNAVAILABLE:
        // handle service down — consider circuit breaker fallback
        break;
      default:
      // handle unexpected error
    }
  }
});
```

---

## 27. Logging & Monitoring

- Use gRPC **interceptors** (gRPC's equivalent of Express middleware) to log every call's method name, duration, and status code uniformly, without repeating this logic in every handler.
- Monitor **per-method latency and error rate** just as you would for REST endpoints — gRPC method names (`PricingService/GetPrice`) become your metric labels instead of REST routes.
- Propagate trace IDs via gRPC **metadata** (gRPC's equivalent of HTTP headers) to maintain cross-service tracing consistency with your REST-based services.

```javascript
// A simple server-side logging interceptor
function loggingInterceptor(methodDescriptor, call) {
  const start = Date.now();
  return {
    ...call,
    sendMessage(message) {
      logger.info({ method: methodDescriptor.path, durationMs: Date.now() - start }, "gRPC call handled");
      call.sendMessage(message);
    },
  };
}
```

---

## 28. Security Considerations

- Always use **TLS** (`grpc.ServerCredentials.createSsl(...)`) in production — the examples above use `createInsecure()` only for local development simplicity.
- Use gRPC **interceptors** to implement authentication (e.g., validating a token passed via metadata) consistently across all methods, similar to REST middleware.
- Since `.proto` files define your exact API surface, be careful about what fields you expose — a strict contract is a security feature (no accidental over-fetching of undocumented fields) but also means you must be deliberate about what each message includes.

---

## 29. Performance Optimization

- gRPC's binary serialization and HTTP/2 multiplexing already provide substantial performance benefits over REST/JSON at high volume — but you should still **reuse the same client/channel** across calls rather than creating a new one per request, to benefit from HTTP/2 connection reuse.
- For very high-throughput scenarios, prefer **streaming RPCs** (e.g., client streaming for batch price lookups) over making many separate unary calls, reducing per-call overhead.
- Profile serialization/deserialization cost for very large messages — while protobuf is generally fast, extremely large single messages may benefit from being split into a stream instead.

---

## 30. Production Best Practices

- Treat `.proto` files as **the API contract**, version-controlled and reviewed with the same rigor as a REST OpenAPI spec — consider a shared repository or package for `.proto` files consumed by multiple services to avoid drift between copies.
- Evolve `.proto` schemas **additively** wherever possible (add new fields with new field numbers; never reuse or renumber existing field numbers) to maintain backward compatibility, mirroring REST's versioning discipline from Module 7.
- Set explicit **deadlines** on every client call, just as you would set timeouts for REST — don't rely on defaults.
- Use a **service mesh or gRPC-aware load balancer** (Section 19) in production Kubernetes deployments to avoid the connection-pinning pitfall.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Using gRPC for public/browser-facing APIs** | Browsers don't natively support raw gRPC without additional tooling (gRPC-Web + a proxy) — REST/JSON remains the better default for public APIs |
| **Renumbering or reusing protobuf field numbers** | Breaks binary compatibility with any client still using the old schema — field numbers must be treated as permanent once used |
| **No deadline set on client calls** | Defeats one of gRPC's built-in resilience advantages; can hang indefinitely just like an unbounded REST call |
| **Naive L4 load balancing for gRPC** | Long-lived HTTP/2 connections can pin all traffic to one server replica, causing uneven load |
| **Duplicating `.proto` files with manual copy-paste across repos** | Leads to schema drift between services over time — use a shared package/registry instead |

```
Protobuf field renumbering (anti-pattern):

  message GetPriceResponse {
    string product_id = 1;
    double price = 2;      // originally field 2
  }

  // LATER, someone "cleans up" the file:
  message GetPriceResponse {
    double price = 1;       // WRONG: reused field number 1,
    string product_id = 2;  // which used to mean product_id!
  }

  Result: any client still using the OLD compiled stubs will
  misinterpret price as product_id and vice versa — a silent,
  serious bug
```

---

## 32. Debugging Tips

- Use tools like `grpcurl` (a command-line client for gRPC, analogous to `curl` for REST) to manually test gRPC endpoints during development and debugging.
- When a call fails, check the **gRPC status code** first (analogous to checking an HTTP status code) to quickly categorize the failure type (client error, not found, deadline exceeded, server error).
- If clients report inconsistent/garbled data, suspect a `.proto` **schema mismatch** between client and server (e.g., one side has stale generated code) before assuming a logic bug.
- For load balancing issues (some pods overloaded, others idle), check whether you're using client-side load balancing or an L7-aware proxy — this is the most common gRPC-specific operational issue.

---

## 33. Interview Questions

### Easy
1. What is gRPC, and what problem was it built to solve?
2. What are Protocol Buffers, and how do they differ from JSON?
3. What is a `.proto` file, and what role does it play?
4. Name gRPC's four call types.
5. Why is gRPC generally not used directly for public browser-facing APIs?

### Medium
6. How does gRPC's use of HTTP/2 provide a performance advantage over typical REST/HTTP1.1 usage?
7. What is a gRPC deadline, and how does it compare to a REST timeout?
8. Explain server streaming RPC with a real-world use case.
9. Why must protobuf field numbers never be reused or renumbered once assigned?
10. When would you choose REST over gRPC for an internal service-to-service call, even though gRPC is generally faster?

### Hard
11. Design the `.proto` contract for a "real-time order tracking" feature using an appropriate gRPC call type, and justify your choice.
12. Explain the gRPC load balancing pitfall with long-lived HTTP/2 connections, and describe two different ways to solve it.
13. How would you evolve a gRPC service's contract to add a new required-feeling field without breaking existing clients still on the old `.proto` version?
14. Compare the debugging experience of a gRPC service versus a REST service, and what tooling differences you'd need to account for.
15. Design a system where a single service exposes BOTH a REST API and a gRPC API, and explain what criteria you'd use to decide which internal callers use which.

---

## 34. Scenario-Based Questions

1. Your `pricing-service` is being hit by 5,000 requests/second internally, and JSON serialization overhead is showing up as a significant chunk of CPU time in profiling. What would you propose, and how would you justify the migration cost?
2. After migrating an internal call from REST to gRPC, you notice that only 1 of your 5 `pricing-service` replicas is receiving nearly all the traffic. Diagnose and fix.
3. A teammate wants to expose your gRPC API directly to the public-facing mobile app "since it's faster." What concerns would you raise, and what would you recommend instead?
4. Your team renumbered a field in a `.proto` file during a "cleanup" PR, and now a downstream service is receiving corrupted-looking data. Walk through your diagnosis process.
5. You need to give `order-service` a live feed of price changes instead of polling `pricing-service` every 2 seconds. Design this using gRPC and justify your choice of call type.

---

## 35. Hands-on Exercises

1. Write a `.proto` file for an "Inventory Service" with a unary `CheckStock` RPC and a server-streaming `SubscribeToStockChanges` RPC.
2. Implement the gRPC server for your `.proto` file from Exercise 1 in Node.js using `@grpc/grpc-js`.
3. Implement a gRPC client that calls `CheckStock` with a 2-second deadline, and handles `NOT_FOUND` and `DEADLINE_EXCEEDED` explicitly.
4. Use `grpcurl` to manually test your server from Exercise 2 without writing a client at all.
5. Write a short comparison table (REST vs gRPC) filled in with your own words for: performance, contract safety, browser compatibility, streaming support, and debugging tooling.

---

## 36. Mini Project

**Build: A Unary gRPC Pricing Service + Client**

1. Implement `pricing-service` with the `.proto` file and `GetPrice` unary RPC from Section 22.
2. Implement `order-service` with the gRPC client wrapper from Section 23, exposing a REST endpoint (`GET /orders/preview-price/:productId`) that internally calls `pricing-service` via gRPC.
3. Add a 3-second deadline to the gRPC call, and handle `DEADLINE_EXCEEDED` and `NOT_FOUND` gracefully with appropriate REST status codes returned to the client.
4. Containerize both services with Docker Compose, as shown in Section 24.

---

## 37. Advanced Project

**Build: Streaming Price Updates + Hybrid REST/gRPC Architecture**

1. Extend the Mini Project: implement the `SubscribeToPriceUpdates` server-streaming RPC in `pricing-service`.
2. Build a small client (could be a simple Node.js script, or a WebSocket bridge in `order-service` for a frontend) that subscribes to live price updates for a product and logs each update as it arrives.
3. Add an asynchronous Kafka event (`price-events`) published alongside the gRPC stream push, consumed independently by a separate `analytics-service` — demonstrating gRPC (sync/streaming) and Kafka (async) coexisting.
4. Add a gRPC logging interceptor (Section 27) recording method name, duration, and status code for every call.
5. Simulate a schema evolution: add a new optional field to `GetPriceResponse` (e.g., `discount_percentage`) using a new field number, and verify that an OLD client (compiled before this change) continues to work correctly, demonstrating protobuf's backward compatibility when done correctly.

---

## 38. Summary

- gRPC is a high-performance RPC framework built on HTTP/2 and Protocol Buffers, purpose-built for internal, strongly-typed, potentially-streaming service-to-service communication.
- Its `.proto` contract-first approach catches many integration errors at build time rather than runtime, unlike loosely-typed JSON REST APIs.
- gRPC supports four call types: unary (like typical REST), server streaming, client streaming, and bidirectional streaming — enabling native real-time data feeds without bolting on separate technology.
- gRPC is generally the wrong choice for public/browser-facing APIs, where REST's universal compatibility remains preferable.
- Production gRPC requires care around TLS, deadline propagation, protobuf schema evolution discipline, and load balancing for its long-lived HTTP/2 connections.

---

## 39. Revision Notes

- gRPC: HTTP/2 + Protocol Buffers, contract-first via `.proto` files.
- Four call types: unary, server streaming, client streaming, bidirectional streaming.
- Deadlines: gRPC's built-in, propagated timeout mechanism.
- Status codes: gRPC's equivalent of HTTP status codes (NOT_FOUND, DEADLINE_EXCEEDED, UNAVAILABLE, etc.).
- Never reuse/renumber protobuf field numbers — evolve additively.
- Best for internal, high-throughput, strongly-typed communication; not ideal for public/browser-facing APIs.
- Load balancing gRPC's persistent HTTP/2 connections requires client-side balancing or an L7-aware proxy/service mesh.

---

## 40. One-Page Cheat Sheet

```
GRPC:                RPC framework on HTTP/2 + Protocol Buffers
.PROTO FILE:          contract-first schema, generates client/server code
CALL TYPES:           unary | server streaming | client streaming | bidirectional
DEADLINE:             built-in, propagated timeout (like REST's timeout, but native)
STATUS CODES:         OK | INVALID_ARGUMENT | NOT_FOUND | DEADLINE_EXCEEDED | UNAVAILABLE | INTERNAL

USE GRPC WHEN:        internal service-to-service, high throughput, need streaming,
                       want strict contract enforcement
USE REST WHEN:        public/browser-facing API, need wide compatibility, simpler tooling

GOLDEN RULES:
  - Never renumber/reuse protobuf field numbers - evolve additively only
  - Always set a deadline on client calls
  - Use TLS in production, never createInsecure()
  - Use client-side load balancing or a service mesh for long-lived HTTP/2 connections
  - Keep .proto files as a single, shared source of truth across services
```

---

**Suggested Next Module:** Module 9 — Event-Driven Architecture (deep dive into events, domain events, event notification vs. event-carried state transfer, and designing systems around asynchronous event flows)
