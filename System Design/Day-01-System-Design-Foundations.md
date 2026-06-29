# DAY 1 — System Design Foundations
### (What is System Design, Requirements, Scalability, Reliability, Availability, Maintainability, Interview Framework)

> **How to use this file**: Read it top to bottom once. Then re-read only the **"How to teach this"** boxes — those are written so you can literally repeat them to another person, word for word, and they will understand. Every section follows: **What → Why → Background → How → Example → Implementation → Trade-offs → Interview Angle**.

---

## TABLE OF CONTENTS — DAY 1

1. What is System Design (and why it's a separate skill from coding)
2. Functional vs Non-Functional Requirements
3. The 4 Pillars: Scalability, Reliability, Availability, Maintainability
4. How to Approach ANY System Design Problem (Step-by-step Framework)
5. A Worked Example using the Framework (Design a Pastebin)
6. Common Mistakes Candidates Make
7. Day 1 Cheat Sheet (for quick revision before interviews)

---

## 1. WHAT IS SYSTEM DESIGN?

### What
System design is the process of defining the **architecture, components, modules, interfaces, and data flow** of a software system to satisfy specified requirements — both **functional** (what it should do) and **non-functional** (how well it should do it: fast, reliable, scalable, secure).

In simple words: **Coding answers "how do I write this function?" System design answers "how do thousands of these functions, running on hundreds of machines, talk to each other without falling apart when 10 million people show up at once?"**

### Why does this exist as a separate discipline?
As a Node.js developer, you already know how to write an Express route that fetches a user from MongoDB and returns JSON. That works perfectly... for 10 users.

Now imagine:
- 10 million users hit that same route every second.
- Your database server's hard disk can only handle 5,000 read operations per second.
- Your single Node.js process can only handle ~10,000 concurrent connections before it runs out of memory.
- Your server is in Mumbpai, but your users are in the US, India, and Australia — and network latency to/from each location is different.
- Your server's hard disk crashes at 3 AM.

None of these problems are solved by "writing better code" in the traditional sense. They're solved by **architectural decisions**: how you split data across multiple databases, how you cache results, how you replicate data to survive crashes, how you route users to the nearest server. **This is system design.**

### Background (the history — why this became a "thing")
In the early days of the web (1995–2005), most apps were small: one web server, one database, hosted on one machine. This is called a **monolith on a single server**.

As companies like Amazon, Google, Facebook grew to hundreds of millions of users, a single machine — no matter how powerful — physically could not handle the load (CPU, RAM, disk I/O, and network bandwidth all have hard physical limits). Engineers had to invent ways to **split work across multiple machines** and keep those machines coordinated, consistent, and available. Concepts like load balancing, database sharding, caching (CDNs, Redis), message queues, and distributed consensus all came out of solving *real* outages and bottlenecks at companies like Amazon (which published the famous "Dynamo" paper in 2007), Google (which published the "MapReduce" and "Bigtable" papers), and LinkedIn (which built Kafka).

System design as an **interview topic** became popular post-2010s because companies realized that knowing how to code a binary search is not the same skill as knowing how to design WhatsApp for a billion users. Big Tech (Google, Amazon, Meta, Microsoft) started asking these questions specifically to test "can this engineer think about systems at scale," which is what senior/staff engineers actually do all day.

### How (the general thought process)
System design is **not** about memorizing one correct diagram. It's about:
1. Clarifying what the system actually needs to do (requirements).
2. Estimating the scale (how many users, how much data, how many requests/sec).
3. Designing a high-level architecture (boxes and arrows: client → load balancer → servers → cache → database).
4. Going deep into 2–3 critical components (e.g., "how exactly does the database avoid becoming a bottleneck?").
5. Identifying bottlenecks and single points of failure, and fixing them.
6. Discussing trade-offs — because **there is no perfect system**, only systems that are good trade-offs for a specific use case.

### Real-world example
- **Netflix**: Doesn't store all movies on one server. Uses CDNs (Open Connect) placed inside ISPs around the world so video is served from a server physically close to you — this is a system design decision to reduce latency and bandwidth cost.
- **WhatsApp**: In 2014, ran on roughly 550 servers to support ~900 million users (a famous example of efficient system design using Erlang's concurrency model) — proving good system design lets you do more with less hardware.

### How to teach this (script you can reuse)
> "Think of coding as building a single brick really well. System design is the architecture of the entire building — how many floors, how the elevators avoid jams during rush hour, how water reaches the 50th floor, and what happens if one floor catches fire. The brick-laying skill doesn't change. But now you have to think about thousands of bricks working together under real-world stress."

### Interview Angle
Interviewers use system design rounds to check: Can you handle ambiguity? Can you ask the right clarifying questions? Do you know the standard building blocks (load balancer, cache, queue, database)? Can you reason about trade-offs instead of pretending there's one right answer? Can you estimate numbers (capacity planning)?

---

## 2. FUNCTIONAL vs NON-FUNCTIONAL REQUIREMENTS

### What
- **Functional Requirements (FR):** What the system *must do*. These are the features. Example for a URL shortener: "User can submit a long URL and get back a short URL." "When someone visits the short URL, they get redirected to the original URL."
- **Non-Functional Requirements (NFR):** How well the system must perform those functions. These describe *qualities*, not features. Example: "The redirect should happen in under 100ms." "The system should support 100 million URLs." "The system should be available 99.99% of the time."

### Why this distinction matters
If you only think about functional requirements, you'll design something that "works" in a demo but collapses in production. If you only think about non-functional requirements, you have no idea what you're actually building.

**The single biggest mistake candidates make in interviews is jumping straight to drawing boxes without first listing FRs and NFRs.** This looks like you're guessing instead of engineering.

### Background
This distinction comes from classical software engineering / requirements engineering (going back to the 1970s–1990s, formalized in standards like IEEE 830). It was originally used for any software project (not just distributed systems) but became central to system design interviews because it forces structured thinking under time pressure.

### How (steps to extract FR/NFR in an interview or real project)
1. **Ask clarifying questions** to the interviewer/stakeholder. Never assume.
   - "Should this support both read and write operations, or is it read-heavy?"
   - "What's the expected number of users / requests per day?"
   - "Do we need real-time updates, or is some delay (eventual consistency) acceptable?"
2. List FRs as simple bullet "the system should do X" statements.
3. List NFRs using the **4 Pillars** (next section) as a checklist: Scalability, Reliability, Availability, Maintainability — plus Latency, Consistency, Security if relevant.
4. **Prioritize.** You cannot optimize for everything (this is a core system design truth — trade-offs are everywhere). Ask: "Of these NFRs, which 2 matter most for this product?" E.g., for a payments system, **consistency and reliability** matter more than low latency. For a social media news feed, **availability and latency** matter more than perfect consistency (it's fine if a like count is 2 seconds stale).

### Example — Designing Instagram
**Functional Requirements:**
- Users can upload photos/videos.
- Users can follow other users.
- Users can see a feed of posts from people they follow.
- Users can like/comment on posts.

**Non-Functional Requirements:**
- Highly available (users should almost never see an error page).
- Low latency feed loading (under 200ms).
- Can scale to handle 500 million daily active users.
- Eventual consistency is acceptable (it's OK if a like appears 1-2 seconds late to other viewers).
- Durable storage (a photo, once uploaded, should never be lost).

Notice: photo upload reliability (never losing data) is critical, but feed consistency (exact real-time count) is not. This kind of prioritization is exactly what interviewers want to hear.

### Implementation Angle (as a Node.js dev, how this maps to real decisions)
Functional requirements map almost directly to your **API endpoints / routes**:
```js
// Functional Requirement: "User can shorten a URL"
app.post('/api/shorten', async (req, res) => {
  const { longUrl } = req.body;
  const shortCode = generateShortCode(); // logic discussed later in Day 7
  await db.urls.insert({ shortCode, longUrl });
  res.json({ shortUrl: `https://short.ly/${shortCode}` });
});

// Functional Requirement: "Visiting short URL redirects to original"
app.get('/:shortCode', async (req, res) => {
  const entry = await db.urls.findOne({ shortCode: req.params.shortCode });
  if (!entry) return res.status(404).send('Not found');
  res.redirect(301, entry.longUrl); // 301 = permanent redirect, cached by browser
});
```
Non-functional requirements map to **architectural decisions surrounding these endpoints** — e.g., "must handle 1000 redirects/sec" means you'll add caching (Redis) in front of the database lookup, and a load balancer in front of multiple Node.js instances. NFRs rarely change your route code; they change *what infrastructure sits around* your code.

### Trade-offs
You cannot maximize every NFR simultaneously:
- More consistency often means **higher latency** (you have to wait for confirmation from multiple servers).
- More availability sometimes means **eventual consistency** (serve a possibly-stale answer rather than no answer).
- More security checks (encryption, validation) add **processing overhead**, slightly increasing latency.

This is why system design has no single "correct" answer — only answers that are well-justified given the *priorities* you (or your interviewer) set.

### Interview Angle
Expect: "Design X." Your **first move**, before drawing anything, should be: "Let me clarify the requirements first." Spend 3–5 minutes listing FR/NFR out loud. This single habit makes you look senior immediately.

### How to teach this (script)
> "Functional requirements are the 'what' — the features a user can see and use. Non-functional requirements are the 'how well' — invisible promises like speed, uptime, and not losing data. A car's functional requirement is 'it drives you from A to B.' Its non-functional requirements are 'it doesn't break down for 10 years, it's safe in a crash, and it doesn't waste fuel.' You can't design a good car — or a good system — without listing both."

---

## 3. THE 4 PILLARS: SCALABILITY, RELIABILITY, AVAILABILITY, MAINTAINABILITY

These four words appear in almost every system design conversation. Interviewers expect you to define them precisely and **not confuse them with each other** (this is one of the most common point-losing mistakes).

---

### 3.1 SCALABILITY

#### What
Scalability is the system's ability to **handle increasing amounts of load** (more users, more data, more requests) by adding resources, **without a redesign**.

#### Why
Every successful product grows. A system designed only for today's load will need a complete rewrite when traffic doubles — that's expensive, risky, and slow. Designing for scalability from the start (or knowing where the "seams" are to scale later) saves the company from a "rewrite under fire" situation, which is one of the most dangerous situations in engineering (rewriting while the current system is also on fire from too much load).

#### Background
The term gained prominence as web companies in the 2000s (Google, Amazon, Yahoo) hit physical limits of single machines. The realization was: **you cannot buy a server powerful enough to serve a billion users — you must use MANY servers together.** This gave birth to "horizontal scaling" as a discipline, distinct from just "buying a bigger machine."

#### How — Two ways to scale:

**A. Vertical Scaling (Scale UP)**
Add more power (CPU, RAM, SSD) to your EXISTING single machine.
- Example: Upgrading your Node.js server from a 2-core, 4GB RAM machine to a 32-core, 256GB RAM machine.
- **Pros:** Simple — no code changes, no distributed systems complexity (no need to worry about syncing data between machines).
- **Cons:** There's a hard physical ceiling (you can't buy infinite RAM/CPU on a single machine). Also a single point of failure — if that one beefy machine dies, EVERYTHING goes down. Cost increases non-linearly (a machine with 2x the specs often costs much more than 2x the price).

**B. Horizontal Scaling (Scale OUT)**
Add MORE machines, and distribute load across them.
- Example: Instead of 1 Node.js server handling 10,000 req/sec, run 10 Node.js servers each handling 1,000 req/sec, with a load balancer distributing requests between them.
- **Pros:** No real ceiling (theoretically add as many machines as you want). No single point of failure (if one server dies, others keep serving). Usually cheaper per unit of capacity (commodity hardware).
- **Cons:** Much more complex. Now you need: a load balancer, a way to keep servers stateless (or share state via a common store like Redis), and you need to think about data consistency across machines if they touch a shared database.

#### Implementation — visualizing both in Node.js terms

**Vertical scaling** — literally no code change. You just deploy the same app to bigger hardware (e.g., move from an AWS `t3.medium` to a `c6i.8xlarge`).

**Horizontal scaling** — requires your app to be **stateless** (very important concept, explained fully on Day 4) so any server can handle any request:
```js
// BAD for horizontal scaling — storing session in server memory
let userSessions = {}; // if this server restarts or another server gets
                        // the next request from this user, the session is GONE

app.post('/login', (req, res) => {
  userSessions[req.body.userId] = { loggedIn: true }; // tied to THIS process only
  res.send('Logged in');
});

// GOOD for horizontal scaling — externalize state to Redis
// Now ANY of your 10 Node.js servers can read this session
const redisClient = require('redis').createClient();

app.post('/login', async (req, res) => {
  await redisClient.set(`session:${req.body.userId}`, JSON.stringify({ loggedIn: true }));
  res.send('Logged in');
});

app.get('/profile', async (req, res) => {
  const session = await redisClient.get(`session:${req.body.userId}`);
  if (!session) return res.status(401).send('Not logged in');
  res.json(JSON.parse(session));
});
```
This is the **core principle** behind horizontal scaling: don't store anything important only in one server's memory/disk. Push shared state to a common backing store (Redis, database) that all your server instances can access.

#### Real-world example
- Netflix horizontally scales thousands of microservice instances across AWS using auto-scaling groups — when traffic spikes (e.g., a popular new show drops), more server instances spin up automatically, and shrink back down when traffic falls, to save cost.

#### Trade-offs
| | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| Complexity | Low | High |
| Cost at small scale | Cheaper | More expensive (need LB, multiple machines) |
| Cost at large scale | Becomes extremely expensive / impossible | More cost-efficient |
| Single point of failure | Yes | No (if done right) |
| Ceiling | Hard physical limit | Practically unlimited |

In real systems: most companies do **both** — scale up a bit, then scale out, then scale up the individual nodes further, then scale out more. It's not "either/or" — it's a spectrum used together.

#### Interview Angle
If asked "how would you scale this system to 10x traffic," the answer is almost always: "First identify the bottleneck (CPU? DB? Network?), then prefer horizontal scaling for stateless components, and use database-specific techniques (replication, sharding — Week 2 topics) for the data layer, since you can't just 'add a server' to scale a single database easily."

#### How to teach this (script)
> "Imagine a single cashier at a supermarket. Vertical scaling is making that one cashier superhuman — faster hands, more arms. There's a limit to how superhuman one person can get. Horizontal scaling is opening 10 more checkout counters with 10 more cashiers. Now you can serve more customers in parallel, and if one cashier goes on a break, the others still work."

---

### 3.2 RELIABILITY

#### What
Reliability is the probability that a system performs its intended function **correctly** over a given period of time, **even when something goes wrong** (hardware failure, software bugs, human error, malicious attacks). A reliable system "fails gracefully" — it doesn't lose data or give wrong answers, even under faults.

#### Why
Hardware fails. Disks crash. Network cables get physically cut. Code has bugs. The question is never "will something fail" — it's "when it fails, does my system handle it correctly, or does it corrupt/lose data and give wrong results?" Reliability is about **correctness under failure**.

#### Background
The formal study of reliability comes from hardware engineering (e.g., NASA spacecraft systems, telecom systems that must work even during disasters) and was adapted into distributed software systems as companies realized that with thousands of servers, **something is ALWAYS failing, all the time** — at Google/Amazon scale, hard disks fail every single day across their fleet. Reliability engineering is about designing systems where individual component failures don't cause overall system failure or incorrect output.

#### How — Techniques to achieve reliability
1. **Redundancy** — Don't rely on a single instance of anything critical. Keep multiple copies of data (replication) and multiple instances of services.
2. **Error detection and correction** — Checksums to verify data isn't corrupted; retries with idempotency for failed network calls.
3. **Graceful degradation** — If a non-critical part fails (e.g., recommendation engine), the core system (e.g., checkout) should still work, just without recommendations.
4. **Automated testing & monitoring** — Catch bugs before users do; catch failures the moment they happen (Day 24 covers this).

#### Implementation Example — Retry logic with idempotency in Node.js
A classic reliability problem: your Node.js server calls a Payment API. The request reaches the payment server, charges the customer, but the response is lost due to a network blip. Your server thinks it failed and retries → customer gets charged TWICE. This is a **reliability bug**.

**Fix: Idempotency keys** (also covered in depth Day 23):
```js
const { v4: uuidv4 } = require('uuid');

async function chargeCustomer(orderId, amount) {
  // Generate the SAME idempotency key every time for the SAME logical operation
  const idempotencyKey = `charge_${orderId}`; // tied to order, not random each time

  const response = await paymentGatewayClient.post('/charge', {
    amount,
    idempotencyKey, // payment provider uses this to detect duplicate requests
  }, {
    headers: { 'Idempotency-Key': idempotencyKey }
  });

  return response.data;
}

// Even if this function is called twice due to a retry,
// the payment gateway recognizes the same idempotencyKey
// and does NOT charge the customer again — it just returns
// the result of the first successful charge.
```
This is a real, production-grade reliability pattern used by Stripe, PayPal, and basically every payment system in the world.

#### Real-world example
- **Amazon S3** is designed for "11 nines" (99.999999999%) durability — meaning if you store 10 million objects, you'd statistically expect to lose one object roughly once every 10,000 years. This is achieved through massive redundancy (data is copied across multiple disks, racks, and data centers).

#### Trade-offs
Higher reliability = more redundancy = more cost (storing 3 copies of data costs 3x the storage) and more engineering complexity (need to keep copies in sync). Companies usually pick reliability targets based on the cost of failure: a banking system needs near-zero data loss tolerance; a "draft auto-save" feature for a blog post can tolerate occasional loss.

#### Interview Angle
"How would you make this reliable?" → Talk about replication, retries+idempotency, checksums/validation, and graceful degradation (circuit breakers — Day 20). Don't confuse this with availability (next section) — that's the #1 trap.

#### How to teach this (script)
> "Reliability is about correctness despite failure. Think of a backup parachute. If the main parachute fails (a fault), the backup ensures the skydiver still lands safely (correct outcome). Reliability isn't about preventing every possible failure — that's impossible. It's about making sure failures don't lead to WRONG or LOST outcomes."

---

### 3.3 AVAILABILITY

#### What
Availability is the percentage of time a system is **operational and able to respond to requests**, measured over a period (usually a year). Expressed as "nines" — e.g., 99.9% ("three nines"), 99.99% ("four nines").

#### Why
Even a perfectly "reliable" system (never gives wrong answers) is useless if it's frequently unreachable. Availability answers: "When a user tries to use the system right now, will they get a response at all?"

#### Background
Telecom companies pioneered "uptime" SLAs (Service Level Agreements) for phone networks decades before the internet — the famous "five nines" (99.999%) standard came from telephone switch reliability requirements, where even a few minutes of downtime a year was considered unacceptable for emergency calls. Internet companies adopted the same language for web services.

#### How — The Math (this gets asked directly in interviews!)

| Availability | Downtime per year | Downtime per month | Downtime per day |
|---|---|---|---|
| 99% ("two nines") | ~3.65 days | ~7.3 hours | ~14.4 minutes |
| 99.9% ("three nines") | ~8.76 hours | ~43.8 minutes | ~1.44 minutes |
| 99.99% ("four nines") | ~52.6 minutes | ~4.32 minutes | ~8.6 seconds |
| 99.999% ("five nines") | ~5.26 minutes | ~25.9 seconds | ~0.86 seconds |

**How to calculate Availability of a system made of multiple components:**

- **If components are in SERIES (request must pass through ALL of them)**, total availability = multiply individual availabilities.
  - Example: Load Balancer (99.99%) → App Server (99.9%) → Database (99.99%)
  - Total = 0.9999 × 0.999 × 0.9999 ≈ **99.88%** — Notice it's LOWER than any individual component! Chaining dependent components always reduces overall availability.

- **If components are in PARALLEL (redundant — only ONE needs to work)**, total availability = `1 - (probability all fail)`.
  - Example: 2 redundant database replicas, each 99% available.
  - Probability BOTH fail simultaneously = (1 - 0.99) × (1 - 0.99) = 0.01 × 0.01 = 0.0001
  - Total availability = 1 - 0.0001 = **99.99%** — Notice it's HIGHER than either individual component! Redundancy increases availability.

**This is the single most important mathematical insight in system design: chaining things in series HURTS availability; adding redundancy in parallel HELPS availability.**

#### Implementation — Health checks (the basic building block of availability)
Load balancers determine availability by continuously checking if servers are alive:
```js
// A simple health check endpoint every Node.js service should expose
app.get('/health', async (req, res) => {
  try {
    // Check critical dependencies are reachable
    await db.ping();        // is the database reachable?
    await redisClient.ping(); // is the cache reachable?
    res.status(200).json({ status: 'healthy' });
  } catch (err) {
    // If a dependency is down, report unhealthy so load balancer
    // stops sending traffic to THIS instance
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});
```
Load balancers (Day 4) ping `/health` every few seconds. If a server fails 2-3 consecutive checks, it's removed from the pool of servers receiving traffic — this is literally how "availability" is engineered at the infrastructure level.

#### Real-world example
- Google's SLA for many cloud services promises 99.95% availability — and if they miss it, customers get service credits. This drives massive engineering investment in redundancy (multi-region deployments, automatic failover).

#### Trade-offs
High availability requires redundancy (multiple servers/regions), which costs money and adds complexity (data must be synced across redundant copies, which connects directly to the CAP theorem — Day 12). There's also a philosophical trade-off: sometimes it's better to show a **slightly stale** cached response (available, but not perfectly up to date) than to show an error page (unavailable but "honest"). Most consumer systems (social media, e-commerce browsing) choose availability over perfect consistency.

#### Interview Angle
Be ready to **calculate** availability percentages and convert them to downtime, and explain series-vs-parallel availability math — this is a very common rapid-fire interview question. Also be ready to explain: **"Reliability vs Availability — what's the difference?"**

**THE KEY DIFFERENCE (memorize this):**
- **Reliability** = Is the system giving CORRECT results, even during failures? (Correctness)
- **Availability** = Is the system RESPONDING right now? (Uptime/Reachability)

A system can be **available but not reliable**: it responds instantly to every request, but sometimes gives you the wrong answer or corrupted data.
A system can be **reliable but not available**: when it does respond, the answer is always correct, but it's down 30% of the time.
**You want both — but they are different and improved by different techniques.**

#### How to teach this (script)
> "Availability is 'can I reach the shop right now?' Reliability is 'when I buy something from this shop, will the product actually work?' A shop open 24/7 but selling broken products is highly available, but unreliable. A shop that sells perfect products but is only open 2 hours a week is reliable, but has low availability. A great business — and a great system — needs both."

---

### 3.4 MAINTAINABILITY

#### What
Maintainability is how easily the system can be **understood, modified, fixed, and extended** by engineers over time — including engineers who didn't originally build it.

#### Why
Systems live for years, sometimes decades. The original team moves on. New features must be added. Bugs must be fixed quickly. If a system is a tangled, undocumented mess, every change becomes slow, risky, and expensive — this is called "technical debt." Companies often lose more money to slow feature delivery from unmaintainable systems than to outages.

#### Background
This concept grew out of software engineering research in the 1980s-90s (the "software crisis" era) when companies realized that **most of the cost of software is NOT writing it — it's maintaining it** for years afterward (studies have repeatedly shown 60-80% of software lifecycle cost is maintenance, not initial development).

#### How — Three sub-qualities of maintainability:
1. **Operability** — How easy is it to keep the system running smoothly in production? (good logging, monitoring, clear runbooks for incidents)
2. **Simplicity** — How easy is it for a new engineer to understand the system? (avoiding unnecessary complexity, clean abstractions, good naming)
3. **Evolvability** — How easy is it to make changes/add features later? (loose coupling between components, so changing one part doesn't break five others)

#### Implementation Example — Designing for maintainability in Node.js
**Bad (low maintainability)** — everything tangled in one file, business logic mixed with HTTP and DB logic:
```js
app.post('/order', async (req, res) => {
  const { userId, items } = req.body;
  let total = 0;
  for (const item of items) {
    const product = await db.query('SELECT * FROM products WHERE id = ?', [item.id]);
    total += product.price * item.qty;
  }
  if (total > 1000) total = total * 0.9; // 10% discount hardcoded here, who knows why
  await db.query('INSERT INTO orders ...');
  await sendEmail(userId, 'Order placed!'); // mixing concerns: HTTP, business logic, DB, email all tangled
  res.json({ total });
});
```

**Good (high maintainability)** — separated into layers (controller → service → repository), each with one responsibility:
```js
// controller (HTTP layer) - only handles request/response
app.post('/order', async (req, res) => {
  const result = await orderService.placeOrder(req.body.userId, req.body.items);
  res.json(result);
});

// service (business logic layer) - the "why" of the business rules
async function placeOrder(userId, items) {
  const total = await pricingService.calculateTotal(items); // discount logic lives HERE, isolated
  const order = await orderRepository.create(userId, items, total);
  await notificationService.sendOrderConfirmation(userId, order); // easy to swap email->SMS later
  return order;
}

// repository (data layer) - only handles DB queries
async function create(userId, items, total) {
  return db.query('INSERT INTO orders (...) VALUES (...)', [...]);
}
```
This is the same functionality, but now: a new engineer can understand each piece independently (simplicity), you can test pricing logic without touching the database (evolvability), and swapping email for SMS notifications doesn't touch order logic at all (loose coupling).

#### Real-world example
- Amazon famously moved from a monolith to microservices partly for maintainability reasons — different teams could independently develop, deploy, and scale their own services (e.g., the "Cart" team didn't need to coordinate releases with the "Recommendations" team), dramatically speeding up feature delivery.

#### Trade-offs
Investing in maintainability (clean architecture, documentation, tests) takes more upfront time and can feel like "slowing down" when shipping a quick feature. But it pays off massively over the system's lifetime. Over-engineering for maintainability too early (e.g., building elaborate microservices for a product with 10 users) is also a real mistake — this is called **premature optimization** or "resume-driven development."

#### Interview Angle
Less commonly tested directly via math (unlike availability), but interviewers DO care if you mention things like "I'd keep this service loosely coupled so we can evolve it independently" — it signals senior-level thinking about the *team and long-term* aspects of system design, not just the technical diagram.

#### How to teach this (script)
> "Maintainability is about the future, not just today. A system might work perfectly on day one, but if it's built like a ball of tangled wires, every new feature added later becomes harder and slower, and bugs become harder to fix without breaking something else. Maintainable systems are like LEGO — clean, separate, swappable blocks. Unmaintainable systems are like a knot — pull one string, and you don't know what else moves."

---

## 4. HOW TO APPROACH ANY SYSTEM DESIGN PROBLEM (THE FRAMEWORK)

This is the **master template**. Memorize this sequence — use it for every single system design problem, in interviews or real projects.

### Step 1: Clarify Requirements (3–5 minutes)
- Ask about functional requirements: "What exact features must this system support?"
- Ask about scale: "How many users? How many requests per second? Read-heavy or write-heavy?"
- Ask about non-functional priorities: "Is strong consistency required, or is eventual consistency fine? What's an acceptable latency?"
- **Explicitly state what's OUT of scope.** ("I'll focus on the core feed mechanism, not the ad-serving system.") This shows you can prioritize.

### Step 2: Back-of-the-Envelope Estimation (3–5 minutes)
Calculate rough numbers — this anchors all later decisions (covered in full mathematical detail on Day 6, but the basic flow is):
- Daily Active Users (DAU) → Requests Per Second (RPS)
- Read vs Write ratio
- Storage needed (data size × number of records × retention period)
- Bandwidth needed (data size × requests per second)

### Step 3: Define the API / Interface (2–3 minutes)
Write down the core API endpoints — this forces you to nail down the functional contract before drawing architecture.
```
POST /shorten   { longUrl } -> { shortUrl }
GET  /:code     -> 301 redirect to longUrl
```

### Step 4: High-Level Design (10–15 minutes)
Draw the big boxes: Client → Load Balancer → App Servers → Cache → Database → (Queue, CDN if relevant). Keep it simple first — a basic working flow — before adding complexity.

### Step 5: Deep Dive into Critical Components (10–15 minutes)
Pick the 1-2 most interesting/critical parts and go deep:
- How exactly does the database scale (sharding strategy)?
- How exactly does the cache stay consistent with the database?
- How do you generate unique IDs across multiple servers?

### Step 6: Identify Bottlenecks & Single Points of Failure (5 minutes)
- "If this one database goes down, does the whole system die? Let's add a replica."
- "If traffic spikes 10x at this one component, what breaks first? Let's add caching/queueing there."

### Step 7: Discuss Trade-offs (ongoing throughout)
For every decision, briefly state the trade-off: "I'm choosing eventual consistency here because availability matters more than perfect real-time accuracy for a like-counter."

### How to teach this (script)
> "Think of it like a doctor's checkup: First, you ask the patient what's wrong and gather history (requirements). Then you take basic vitals — height, weight, blood pressure (estimation/scale). Then you do a general physical exam (high-level design). Then, if something seems off, you order a specific deep test — an X-ray, a blood test (deep dive). Finally, you discuss the diagnosis and trade-offs of treatment options with the patient (trade-offs). You never jump straight to surgery (a detailed solution) without this process — and neither should you jump straight to architecture diagrams without requirements."

---

## 5. WORKED EXAMPLE — APPLYING THE FRAMEWORK TO "DESIGN A PASTEBIN" (like pastebin.com)

This is a classic easy/medium interview question — let's run through the framework quickly so you see it in action (we'll do a FULL deep system, the URL Shortener, on Day 7).

**Step 1 — Requirements:**
- FR: User can paste text and get a unique URL. Anyone with the URL can view the paste. Optionally, pastes expire after a set time.
- NFR: High availability (reads should almost never fail). Low latency reads. Durability (a paste shouldn't randomly disappear before its expiry).

**Step 2 — Estimation:**
- Assume 1 million new pastes/day → ~12 writes/sec average.
- Assume reads are 10x writes (people share links, others view) → ~120 reads/sec average.
- Average paste size: 10 KB → 1 million × 10KB = 10 GB/day of new storage.

**Step 3 — API:**
```
POST /paste   { content, expiryTime? } -> { pasteId, url }
GET  /paste/:pasteId -> { content }
```

**Step 4 — High-Level Design:**
Client → Load Balancer → Node.js App Servers → Cache (Redis, for hot/popular pastes) → Database (stores paste content, keyed by pasteId) → Object Storage (S3, if pastes can be very large, store content there and just keep metadata in DB).

**Step 5 — Deep Dive:**
How do we generate a unique `pasteId`? (This connects to Day 23's Unique ID Generation topic — options: random string + collision check, or a counter-based approach like base62 encoding of an auto-incrementing ID).

**Step 6 — Bottlenecks:**
Database could be a bottleneck for reads at scale → solved by adding a Redis cache in front (cache-aside pattern, Day 17) since pastebin reads are extremely read-heavy and pastes, once created, never change (immutable data is the easiest thing in the world to cache).

**Step 7 — Trade-offs:**
We chose eventual consistency isn't even a concern here since pastes are immutable once written — no update conflicts possible. This simplifies the whole design enormously, which is worth explicitly calling out (recognizing when a hard problem is actually easy due to the nature of the data is a senior-level insight).

---

## 6. COMMON MISTAKES CANDIDATES MAKE (Avoid These)

1. **Jumping straight into a diagram** without asking clarifying questions or stating requirements. (Always do Step 1 first.)
2. **Over-engineering from the start** — designing for Google-scale (billions of users) when the interviewer just wants a working design for a reasonable scale. Always estimate scale first, then match complexity to it.
3. **Confusing Reliability and Availability** — using the words interchangeably. (Reliability = correctness, Availability = uptime.)
4. **Not discussing trade-offs** — presenting a design as if it's the ONLY correct answer, instead of acknowledging "I chose X over Y because of Z priority."
5. **Ignoring the data layer** — spending all the time on app servers/APIs and glossing over how the database will actually scale, which is usually the hardest and most important part.
6. **Silence** — not thinking out loud. Interviewers are evaluating your thought PROCESS, not just the final diagram. Narrate your reasoning constantly.
7. **Not estimating numbers** — saying "we'll add a cache" without ever calculating whether the load actually requires one.

---

## 7. DAY 1 CHEAT SHEET (For Quick Revision)

```
SYSTEM DESIGN = Coding (one brick) vs Architecture (the whole building under stress)

FR  = WHAT the system does (features)
NFR = HOW WELL it does it (speed, uptime, scale, security)

SCALABILITY     = handle MORE load by adding resources
  Vertical   = bigger machine (simple, has a ceiling, single point of failure)
  Horizontal = more machines (complex, no ceiling, needs stateless design)

RELIABILITY     = CORRECT results even when things fail (redundancy, retries, idempotency)
AVAILABILITY    = system RESPONDS when asked (uptime %, measured in "nines")
  - Series components MULTIPLY availability (gets worse)
  - Parallel/redundant components INCREASE availability (gets better)
MAINTAINABILITY = easy to understand, fix, and extend over time (operability,
                   simplicity, evolvability)

FRAMEWORK FOR ANY SYSTEM DESIGN PROBLEM:
1. Clarify Requirements (FR + NFR + scope)
2. Estimate Scale (users, RPS, storage, bandwidth)
3. Define API/Interface
4. High-Level Design (boxes and arrows)
5. Deep Dive on critical components
6. Find & Fix Bottlenecks / Single Points of Failure
7. Discuss Trade-offs throughout

GOLDEN RULE: There is no single "correct" system design — only well-justified
trade-offs for a SPECIFIC set of requirements and priorities. Always say WHY.
```

---

### What's next (Day 2 preview)
Tomorrow we go one level below this — **how the internet itself actually works**: client-server architecture, what really happens when you type a URL into a browser, DNS, HTTP/HTTPS in full depth (every important header, status code, and method), and TCP vs UDP. This is the plumbing that every system design diagram's arrows actually represent — once you know what an "arrow" between two boxes really means at the network level, every future diagram becomes much more meaningful.

**Say "Day 2" whenever you're ready.**
