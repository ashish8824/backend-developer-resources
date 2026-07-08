# Module 29 — System Design Interview Questions

> **Microservices Masterclass** | Level: Expert | Track: Node.js Backend Engineering
> Prerequisite: Module 1–28 (this module is an interview-format application of the entire masterclass)
> Next Module: Module 30 — Production Best Practices & Interview Masterclass

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

Module 28 gave you complete project specifications to build. This module gives you the **interview performance layer** on top of that knowledge: full, worked walkthroughs of six of the most commonly-asked system design interview questions — Design Swiggy (food delivery), Design Uber, Design WhatsApp, Design Netflix, Design Amazon, and Design a Payment Gateway. Each walkthrough follows a repeatable, interview-ready structure: clarify requirements, estimate scale, define the API/data model, sketch the architecture, then go deep on the two or three hardest sub-problems — exactly the flow a strong candidate follows in a real 45-60 minute interview.

This module is not new architectural content — every pattern referenced here was taught in Modules 1-27. It's about **performing** that knowledge fluently, under interview time pressure and interviewer follow-up questions.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Apply a repeatable, structured framework to any system design interview question.
- Correctly scope a system design interview's time budget across requirements, high-level design, and deep dives.
- Walk through complete, interview-quality answers for six of the most frequently-asked system design questions.
- Anticipate and confidently answer common interviewer follow-up/probing questions for each.
- Recognize which of this masterclass's patterns to reach for, quickly, under interview time pressure, for a novel system design prompt not covered explicitly here.

---

## 3. Problem Statement

A candidate who deeply understands Sagas, CQRS, and Event Sourcing can still fail a system design interview by: spending 20 of their 45 minutes drawing boxes without ever addressing the interviewer's actual concern, failing to quantify scale ("how many requests per second are we actually talking about?"), or going deep on an irrelevant detail while never reaching the one or two sub-problems the interviewer specifically wanted explored (e.g., "how do you handle surge pricing consistency" in an Uber design). This module directly addresses the **interview performance** gap — structure, time management, and knowing which sub-problems each classic question is really testing.

---

## 4. Why This Concept Exists

This module exists because **system design interviews test a distinct skill from architectural knowledge alone: structured communication under ambiguity and time pressure.** An interviewer is evaluating not just "do you know what a Saga is," but "can you drive a design conversation, make reasonable assumptions explicit, prioritize your time toward the hardest sub-problems, and defend trade-offs when pushed." This is a learnable, practiceable skill, and having worked through several classic questions in advance — recognizing "oh, this is fundamentally a Trip-Matching problem, I know exactly how to structure this" — meaningfully reduces the cognitive load of performing well in the room.

---

## 5. Historical Background

The "system design interview" as a distinct, widely-recognized interview format solidified across the tech industry through the 2010s, growing directly alongside the industry's shift toward microservices and distributed systems as the default architecture for any company operating at meaningful scale. Companies like Google, Amazon, and later a wave of well-funded startups adopted this format specifically because traditional coding interviews (solving an algorithm on a whiteboard) don't test the skill of designing a real, large-scale, multi-component system — exactly the skill this masterclass has built module by module. Publicly available resources (engineering blogs from Uber, Netflix, Amazon; books like "Designing Data-Intensive Applications") became the de facto shared reference material candidates and interviewers alike draw from — this module distills exactly that shared body of knowledge into the specific six questions asked here.

---

## 6. Real-World Analogy

**Analogy: A Doctor's Differential Diagnosis Process, Under Time Pressure**

A skilled doctor facing a patient with an ambiguous complaint doesn't randomly order every possible test — they follow a **structured process**: ask clarifying questions first (narrowing the possibility space), form a working hypothesis, then run the **specific** tests that would confirm or rule out the most likely explanations, all within a limited appointment window. A system design interview rewards the same disciplined process: clarify the actual requirements first, form a reasonable high-level design, then spend your limited time on the **specific** sub-problems that matter most for this particular "patient" (this particular system), rather than exhaustively covering every possible detail shallowly.

---

## 7. Technical Definition

> A **System Design Interview Walkthrough** is a structured response to an open-ended system design prompt, following a repeatable sequence: **(1) Clarify Requirements & Scope**, **(2) Estimate Scale**, **(3) Define the Core API/Data Model**, **(4) High-Level Architecture**, **(5) Deep Dive** on the 2-3 hardest sub-problems, and **(6) Discuss Trade-offs** — applying this masterclass's specific patterns (Modules 1-27) as justified, concrete answers within this structure.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Requirements Clarification** | The interview's opening phase: narrowing scope, confirming assumptions, before designing anything |
| **Back-of-the-Envelope Estimation** | Quick, approximate scale calculations (requests/sec, storage volume) informing design decisions |
| **High-Level Design (HLD)** | The initial, broad-strokes architecture sketch before going deep on any one part |
| **Deep Dive** | Detailed exploration of a specific, hard sub-problem the interviewer is most interested in |
| **Trade-off Discussion** | Explicitly articulating what a design choice gains and gives up, rather than presenting it as the only correct answer |

---

## 9. Internal Working: The Repeatable Interview Framework

Apply this same six-step structure to any system design prompt, allocating your time roughly as shown:

1. **Clarify Requirements (5 min):** What are the core features? What's explicitly out of scope? Read-heavy or write-heavy? What does "scale" mean here (users, requests/sec)?
2. **Estimate Scale (3-5 min):** Rough numbers — e.g., "100M users, 10% active daily, average 5 requests per session → roughly X requests/second at peak." This directly informs later decisions (Module 24's scaling, Module 14's Polyglot Persistence).
3. **Define Core API & Data Model (5 min):** Sketch the 3-5 most important endpoints/commands and the core entities (Module 4's Bounded Contexts, applied quickly).
4. **High-Level Architecture (10 min):** Draw the main services, their boundaries (Module 5), and how they communicate (Module 6) — broad strokes, not deep detail yet.
5. **Deep Dive (15-20 min):** The interviewer will usually steer you here, or you should proactively pick the 1-2 hardest sub-problems (e.g., "how do you handle a Saga for this checkout" or "how do you scale the read path"). This is where most of your score comes from.
6. **Trade-offs & Wrap-up (5 min):** Explicitly name what you'd reconsider with more time, and the trade-offs of your key decisions.

---

## 10-16: Applying the Framework — Six Worked Examples

Given this module's interview-format nature, Sections 10 through 16 are structured as **six compact worked examples**, each following the Section 9 framework, rather than one extended narrative.

---

### Design Swiggy / Design Uber Eats (Food Delivery)

**Clarify:** Customers order food from restaurants; drivers deliver it. Focus on order lifecycle and driver matching.
**Scale:** ~10M daily active users, ~1M orders/day → ~12 orders/sec average, spiking 10x at meal times.
**Core Entities:** Restaurant, MenuItem, Order, Driver, DeliveryAssignment.
**Architecture:** Restaurant, Order, Delivery-Matching, Payment, Notification services — directly Module 28's Food Delivery project.
**Deep Dive (the interviewer's real interest):** The **Order state machine and Choreographed Saga** (Module 15) — walk through PLACED → RESTAURANT_ACCEPTED → DRIVER_ASSIGNED → PICKED_UP → DELIVERED, each transition an event (Module 9), with explicit handling for "what if no restaurant accepts within N minutes" (timeout + fallback, Module 18) and "what if no driver is available" (re-broadcast or surge incentive).
**Trade-offs:** Choreography (chosen) vs. Orchestration (Module 15) — justify why choreography fits this naturally multi-party, long-running process better than a rigid central orchestrator.

---

### Design Uber / Design Lyft (Ride-Sharing)

**Clarify:** Riders request trips; drivers are matched; pricing may surge with demand.
**Scale:** ~5M daily rides, geographically distributed, extremely latency-sensitive matching (sub-second).
**Core Entities:** Rider, Driver, Trip, Fare.
**Architecture:** Rider, Driver, Trip-Matching, Pricing, Payment, Trip-Tracking — Module 28's Ride-Sharing project.
**Deep Dive:** **Geospatial driver matching at scale** — discuss partitioning the map into a grid or using a geospatial index (e.g., Uber's publicly-documented H3 hexagonal grid system) so matching queries don't require scanning every driver globally; and **surge pricing consistency** — a fast-changing, high-concurrency shared value, discuss Redis-backed atomic counters and the deliberate CAP trade-off (Module 21) of how fresh this value needs to be.
**Trade-offs:** gRPC (Module 8) vs REST for the latency-critical matching call — justify gRPC's performance benefit here specifically.

---

### Design WhatsApp / Design a Chat Application

**Clarify:** 1:1 and group messaging, delivery/read receipts, online presence.
**Scale:** Billions of messages/day, extremely high concurrent connection count (not just request throughput).
**Core Entities:** User, Conversation, Message, PresenceStatus.
**Architecture:** User, Messaging (WebSocket-based), Presence, Media — Module 28's Chat project.
**Deep Dive:** **Message delivery guarantees and connection routing across many server instances** — walk through the Redis Pub/Sub backplane pattern (Module 28, Section 22) solving "which specific instance holds this user's live connection," and discuss at-least-once delivery with client-side deduplication (Module 9's idempotency principle, applied to message IDs) for offline/reconnect scenarios.
**Trade-offs:** WebSockets (chosen, for genuine bidirectional real-time need) vs. long-polling/Server-Sent Events — justify why full bidirectional WebSockets fit chat specifically, unlike a simpler one-directional notification feed.

---

### Design Netflix / Design YouTube (Video Streaming)

**Clarify:** Users upload/browse/watch video; focus on the upload-to-playback pipeline and content delivery at scale.
**Scale:** Massive read (viewing) volume, much lower write (upload) volume, huge binary data sizes.
**Core Entities:** Video, TranscodingJob, User, ViewingHistory.
**Architecture:** Upload, Transcoding-Pipeline, Content-Catalog, Playback, Recommendation — Module 28's Video Streaming project.
**Deep Dive:** **The asynchronous transcoding pipeline** (Module 9's event-driven stages) and **content delivery at scale** — discuss why actual video bytes belong in object storage + a CDN (Module 28's key distinction), never in a traditional service database, and how the Content-Catalog's CQRS read model (Module 16) serves massive read traffic independently of the write-side upload/transcoding pipeline.
**Trade-offs:** Storing multiple pre-transcoded resolutions (more storage cost, faster playback) vs. transcoding on-demand (less storage, higher latency per request) — a real, defensible trade-off to discuss.

---

### Design Amazon (E-Commerce at Scale)

**Clarify:** Product browsing, cart, checkout, order fulfillment — effectively Module 27's ShopFast, at a much larger scale.
**Scale:** Hundreds of millions of products, extreme read/write skew (browsing vastly exceeds ordering), massive seasonal spikes (Black Friday).
**Core Entities:** Product, Cart, Order, Payment, Inventory.
**Architecture:** Directly Module 27's ShopFast architecture.
**Deep Dive:** **Handling a 20x Black Friday traffic spike** (directly Module 24's scenario) — walk through identifying the actual bottleneck (likely the database under a naively-scaled application layer), read replicas for Catalog, and the Saga-based checkout (Module 15) remaining correct even under extreme load via idempotency and proper concurrency control (Module 7, 17).
**Trade-offs:** Strong consistency for inventory counts (avoiding overselling) vs. availability during extreme load — discuss the "reserve, don't immediately deduct" semantic lock pattern from Module 15.

---

### Design a Payment Gateway

**Clarify:** Processing card/bank payments for merchants, handling charges, refunds, and reconciliation, with extremely high correctness requirements.
**Scale:** Potentially tens of thousands of transactions/second for a major provider; correctness matters far more than raw throughput.
**Core Entities:** Merchant, Transaction, Charge, Refund, Ledger.
**Architecture:** Directly Module 28's Banking System project, adapted — Transaction/Ledger Service (Event-Sourced, Module 17), Fraud-Detection (async, Module 9), Reconciliation.
**Deep Dive:** **Idempotency for charge requests** (Module 7's idempotency key pattern is essentially mandatory here — a network retry must never double-charge a customer) and **the Ledger's Event Sourcing design** (Module 17) ensuring every financial state change is a permanent, auditable, replayable event, with strict optimistic concurrency control preventing lost updates under concurrent requests.
**Trade-offs:** Synchronous charge confirmation (customer waits, but knows immediately) vs. asynchronous processing with a webhook callback (faster initial response, more complex client integration) — a real trade-off major payment providers (Stripe) have made differently for different API surfaces.

---

## 17. Database Interaction

Across all six questions, the same Module 14 principle recurs: choose persistence per service based on actual access patterns — relational/ACID for Orders/Payments/Ledgers, document stores for flexible Catalogs, geospatial-capable stores for location-based matching (Uber), object storage for binary media (Netflix), and Redis for fast-changing ephemeral state (Presence, Pricing) — a strong interview signal is explicitly naming *why* each choice fits, not just naming a technology.

---

## 18. Failure Scenarios

A strong interview answer always addresses "what happens when X fails" for at least one critical dependency per design — e.g., "what if the Payment provider times out during Uber's post-ride charge" (Module 18's Circuit Breaker + Retry with idempotency) or "what if a video transcoding job fails" (Module 18's retry-with-backoff, then flag for manual review).

---

## 19. Scalability Discussion

Each of these six questions has a distinct scaling bottleneck worth naming explicitly (mirroring Module 28, Section 19): Swiggy/Uber Eats — driver/restaurant matching under meal-time spikes; Uber — geospatial query performance; WhatsApp — connection count, not request throughput; Netflix — CDN/storage cost and transcoding compute; Amazon — database read scaling under extreme skew; Payment Gateway — correctness-preserving throughput (never sacrificing consistency purely for raw scale).

---

## 20. High Availability Considerations

Payment Gateway and Amazon's checkout path demand the strictest HA guarantees among these six (financial correctness, revenue-critical); WhatsApp's HA concern is specifically about the Pub/Sub backplane's availability (Module 28); Netflix's HA concern is largely delegated to CDN infrastructure's own redundancy, a distinct pattern from the other five.

---

## 21. CAP Theorem Implications

This module is a excellent venue to demonstrate CAP fluency across genuinely different contexts: firmly argue Consistency-favoring choices for Payment Gateway/Amazon inventory, and comfortably defend Availability-favoring choices for Netflix recommendations/WhatsApp presence — the ability to correctly place each of these six systems' key data on the CAP spectrum, and explain why, is one of the highest-value signals in a system design interview.

---

## 22. Node.js Implementation

Given this module's interview-format focus, "implementation" here means being ready to sketch a **short, illustrative code snippet** if asked — interviewers occasionally probe with "can you sketch the actual charge endpoint's idempotency handling?" Reuse Module 7's idempotency middleware pattern directly:

```javascript
// A quick, interview-whiteboard-appropriate sketch for the
// Payment Gateway's idempotent charge endpoint
app.post("/v1/charges", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"];
  const existing = await redis.get(`idem:${idempotencyKey}`);
  if (existing) return res.status(200).json(JSON.parse(existing));

  const result = await processCharge(req.body); // the actual business logic
  await redis.set(`idem:${idempotencyKey}`, JSON.stringify(result), { EX: 86400 });
  res.status(201).json(result);
});
```

---

## 23. Express.js Examples

Being ready to sketch the **Saga orchestrator skeleton** (Module 15) quickly is similarly valuable for the Swiggy/Uber Eats or Amazon deep dives:

```javascript
async function placeOrderSaga(orderInput) {
  await reserveStock(orderInput.items);      // step 1
  try {
    await chargePayment(orderInput.payment); // step 2
  } catch (err) {
    await releaseStock(orderInput.items);    // compensation
    throw err;
  }
  await confirmOrder(orderInput);            // step 3
}
```

---

## 24. Docker Examples

Interviewers rarely ask for actual YAML in a system design interview, but being ready to mention "each service is containerized and deployed independently via Kubernetes with its own CI/CD pipeline" (Modules 19-20, 26) as a one-line answer to "how would you deploy this" demonstrates full-stack architectural fluency beyond just the application logic.

---

## 25. Kafka/Redis Integration

Across all six designs, correctly naming **where** Kafka (durable, multi-consumer events) versus Redis (fast, ephemeral state/pub-sub) fits is a strong signal: Kafka for Order/Trip/Transaction event streams feeding multiple independent consumers; Redis for Presence, surge-pricing counters, rate limiting, and idempotency key storage.

---

## 26. Error Handling

A recurring, high-value interview move: explicitly stating your **fallback policy** per dependency (Module 18) — "Payment has no fallback, it must fail visibly; Recommendations degrades to a generic popular-items list" — demonstrates the tiered-criticality thinking from Module 27 without needing to be prompted.

---

## 27. Logging & Monitoring

Briefly mentioning "I'd instrument RED metrics and distributed tracing across all these services, with tighter alerting thresholds on the payment and matching paths specifically" (Modules 21-23, 27) is a fast, high-value way to signal production-mindedness even within a time-constrained interview.

---

## 28. Security Considerations

For Payment Gateway specifically, proactively mentioning PCI-DSS-adjacent concerns (never storing raw card numbers, tokenization, Module 25's least privilege for any service touching payment data) demonstrates awareness beyond this masterclass's core scope, which interviewers specifically probing fintech roles will value.

---

## 29. Performance Optimization

For Netflix/YouTube, proactively mentioning CDN edge caching and adaptive bitrate streaming (serving different quality levels based on client bandwidth) shows awareness of the domain's specific performance concerns beyond generic backend scaling.

---

## 30. Production Best Practices

Across all six, the single most valuable meta-practice is **narrating your trade-offs out loud** as you design — "I'm choosing eventual consistency here because X, but I'd reconsider if Y" — since this is literally what Module 27's entire integration philosophy trains you to do, and it's exactly what separates a strong interview performance from a merely competent one.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem in an Interview |
|---|---|
| **Diving into deep implementation detail before establishing the high-level design** | Wastes limited time; interviewers want to see structured thinking first |
| **Never asking clarifying questions** | Signals an inability to handle real-world ambiguity, a core skill being tested |
| **Applying the same architecture to every question regardless of domain** | Reveals memorization rather than genuine understanding (directly Module 28's warning) |
| **Ignoring the interviewer's specific follow-up questions to continue your planned narrative** | Interviewers steer toward what they want to evaluate — ignoring this signals poor collaboration |
| **Never stating trade-offs, presenting your design as the only correct answer** | Real systems always involve trade-offs; failing to acknowledge this is a red flag |

---

## 32. Debugging Tips

If you find yourself stuck mid-interview, fall back to this module's Section 9 framework explicitly: "Let me step back and make sure I've got the scale right" or "Let me reconsider the data model before going further" are perfectly acceptable, even impressive, recovery moves — interviewers value visible, structured thinking over silent uncertainty.

---

## 33. Interview Questions

(This entire module IS the interview question set — six fully worked examples above. Practice questions for self-testing:)

1. Walk through designing Swiggy/Uber Eats from scratch, in under 45 minutes, out loud.
2. Walk through designing Uber's driver-matching system, focusing specifically on geospatial scaling.
3. Walk through designing WhatsApp's message delivery guarantee system.
4. Walk through designing Netflix's video upload-to-playback pipeline.
5. Walk through designing Amazon's checkout flow under a 20x Black Friday spike.
6. Walk through designing a Payment Gateway's idempotent charge and Event-Sourced ledger.

---

## 34. Scenario-Based Questions

1. Mid-interview, the interviewer says "actually, let's ignore payment for now — focus entirely on how you'd handle 10x more concurrent WebSocket connections for your Chat design." How do you pivot smoothly?
2. You've spent 25 minutes on high-level design and have 15 minutes left with no deep dive yet. How do you recover?
3. The interviewer challenges your choice of eventual consistency for a specific piece of data. How do you defend it, or gracefully revise your position if their challenge reveals a genuine gap?
4. You don't know a specific technology the interviewer mentions (e.g., a specific geospatial indexing algorithm). How do you handle this gracefully while still demonstrating strong underlying judgment?

---

## 35. Hands-on Exercises

1. Practice Section 9's six-step framework out loud, with a timer, for each of the six worked examples in this module.
2. Record yourself (or practice with a peer) walking through "Design Uber" in exactly 45 minutes, then review where your time actually went.
3. For each of the six designs, write down the ONE deep-dive sub-problem you'd proactively steer toward if the interviewer didn't specify one.
4. Pick a system NOT covered in this module (e.g., "Design Instagram" or "Design a URL Shortener") and apply the same framework independently.
5. Write out explicit trade-off statements (Section 30's practice) for each of your six designs' most significant decisions.

---

## 36. Mini Project

**Practice: Full Timed Walkthroughs**

Conduct full, timed (45-minute) walkthroughs of at least 3 of this module's six questions, either solo (narrating out loud and recording yourself) or with a peer acting as interviewer, providing follow-up questions and pushback on your trade-offs.

---

## 37. Advanced Project

**Practice: Novel System Design Under Pressure**

Apply this module's framework to THREE system design questions not explicitly covered in this module (e.g., Design Instagram, Design a URL Shortener, Design a Ticket-Booking system like BookMyShow/Ticketmaster), each within a strict 45-minute time limit, and write a brief self-critique afterward: where did you spend too much time, what deep dive did you choose and was it the right one, and what trade-offs did you fail to mention that you should have.

---

## 38. Summary

- System design interviews test structured communication and prioritization under ambiguity, not just architectural knowledge — this module provided a repeatable six-step framework (clarify, estimate, model, high-level design, deep dive, trade-offs) to apply to any prompt.
- Six fully worked examples (Swiggy/Uber Eats, Uber, WhatsApp, Netflix, Amazon, Payment Gateway) each demonstrated this framework, directly reusing Modules 1-28's patterns matched to each domain's specific stress points.
- The highest-value interview behaviors are: asking clarifying questions early, proactively identifying and deep-diving the hardest sub-problem, and explicitly narrating trade-offs rather than presenting a design as the only correct answer.
- Practicing timed, out-loud walkthroughs — including for novel questions not covered here — is the most effective way to convert this module's content into genuine interview readiness.

---

## 39. Revision Notes

- Six-step framework: Clarify → Estimate Scale → Core API/Data Model → High-Level Design → Deep Dive → Trade-offs.
- Each classic question has a "real" deep dive the interviewer wants: Swiggy/Uber Eats (Choreographed Saga/state machine), Uber (geospatial matching + surge pricing consistency), WhatsApp (connection routing + delivery guarantees), Netflix (async transcoding pipeline + CDN), Amazon (scaling under spike + inventory consistency), Payment Gateway (idempotency + Event-Sourced ledger).
- Always narrate trade-offs explicitly — this is the single highest-value interview behavior.
- Recover gracefully from being stuck by explicitly revisiting the framework's earlier steps.

---

## 40. One-Page Cheat Sheet

```
FRAMEWORK:  Clarify (5m) -> Estimate Scale (5m) -> Core API/Model (5m)
            -> High-Level Design (10m) -> Deep Dive (15-20m) -> Trade-offs (5m)

SWIGGY/UBER EATS:  Choreographed Saga - multi-party order state machine
UBER:               Geospatial matching + surge pricing consistency (Redis)
WHATSAPP:           WebSockets + Redis Pub/Sub backplane + delivery guarantees
NETFLIX:            Async transcoding pipeline (events) + CDN + CQRS reads
AMAZON:             Scaling under spike (Mod 24) + inventory consistency (Saga)
PAYMENT GATEWAY:    Idempotency keys (mandatory) + Event-Sourced Ledger (Mod 17)

GOLDEN RULES:
  - ALWAYS clarify scope and scale BEFORE designing
  - Proactively identify and deep-dive the domain's ACTUAL hardest sub-problem
  - ALWAYS narrate trade-offs explicitly - never present one answer as the only answer
  - Match patterns to the SPECIFIC domain's stress points (Mod 28) - never a generic template
  - Recover gracefully - revisiting the framework mid-interview is a STRENGTH, not a weakness
```

---

**Suggested Next Module:** Module 30 — Production Best Practices & Interview Masterclass (200+ interview questions, scenario-based questions, architecture discussions, common mistakes, debugging strategies, and production checklists)
