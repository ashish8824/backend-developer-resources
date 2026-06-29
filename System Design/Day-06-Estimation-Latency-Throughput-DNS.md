# DAY 6 — Estimation and the Metrics That Matter
### (DNS Deep Dive Continued — Failure & Propagation, Latency vs Throughput, Back-of-the-Envelope Estimation)

> **Why this day matters:** This is the day that turns you from someone who can "describe" a system into someone who can actually **size** one. Every real system design interview expects you to do quick math in your head — how many requests per second, how much storage, how much bandwidth — within the first 5 minutes, BEFORE you draw a single box. Today gives you that exact skill, plus closes out two important concepts: DNS from an operational/failure point of view, and the Latency vs Throughput distinction (the second "commonly confused pair," after Day 1's Reliability vs Availability).

---

## TABLE OF CONTENTS — DAY 6

1. DNS Revisited — Failure Scenarios & Operational Gotchas
2. Latency vs Throughput
3. Back-of-the-Envelope Estimation — The Numbers Every Engineer Should Know
4. The Estimation Framework — Step by Step, With a Full Worked Example
5. Common Estimation Pitfalls
6. Day 6 Cheat Sheet

---

## 1. DNS REVISITED — FAILURE SCENARIOS & OPERATIONAL GOTCHAS

You learned the mechanics of DNS in full on Day 2 (the resolution chain, record types, TTL). Today, we look at DNS from a different angle: **what goes wrong, in real production systems, and what an engineer actually does about it.** This is exactly the kind of "real-world operational knowledge" that separates a candidate who memorized definitions from one who has actually thought about running systems.

### What — DNS as a Single Point of Failure
### Why this is worth revisiting
On Day 1, you learned that availability calculations treat chained ("in series") components as multiplying their failure probabilities together — and DNS sits squarely in that chain for EVERY single request to your system. If DNS is unreachable or misconfigured, it doesn't matter how perfectly engineered the rest of your system is — users can't even find your server's address to begin talking to it. This makes DNS one of the most under-appreciated single points of failure in real architectures.

### Background — real incidents that shaped how the industry thinks about this
Multiple major outages across the industry's history have been caused not by application bugs, but by DNS misconfigurations or DNS provider outages — a popular DNS provider going down can simultaneously take down many seemingly unrelated companies that all happened to depend on that same provider, even though each company's actual application servers were running perfectly fine the entire time. This pattern repeats often enough that "DNS is involved" has become something of an industry in-joke whenever something mysteriously breaks — but the underlying lesson is serious: DNS is real infrastructure, with real failure modes, and deserves real redundancy planning.

### How — The operational practices that address this

**1. Use multiple, redundant authoritative nameservers** (you saw the `NS` record type on Day 2) — never rely on a single nameserver instance; reputable DNS providers run redundant infrastructure across multiple physical locations specifically so one location failing doesn't take your domain offline.

**2. Plan TTL strategically around migrations** (introduced briefly Day 2, expanded here): If you're about to change an IP address (server migration, cloud provider switch, disaster recovery failover), you should LOWER your DNS TTL well in advance of the change (e.g., from a default of 3600 seconds down to 60 seconds, days before the actual migration). This way, when you make the switch, resolvers worldwide pick up the new IP quickly, instead of some fraction of users being stuck for up to an hour (or whatever the old TTL was) hitting a now-decommissioned server.

**3. Use Anycast DNS for resilience and speed**: Major DNS providers (Cloudflare, AWS Route53, Google Cloud DNS) use a technique called **Anycast**, where the SAME IP address is announced from many physically different locations worldwide, and standard internet routing automatically sends each user's query to whichever announcing location is network-topologically closest to them. This simultaneously improves speed (closer = faster) AND resilience (if one location goes down, traffic automatically reroutes to another, without any DNS record needing to change at all).

**4. Have a "DNS failover" health-check setup for critical systems**: Some DNS providers can perform their OWN health checks against your servers (similar in spirit to the load balancer health checks from Day 4, but happening at the DNS layer instead) and automatically stop returning the IP address of an unhealthy server/region, instead returning a healthy one — useful for multi-region disaster recovery setups (a concept we'll explore fully in Week 4's High Availability topics).

### Real-world example
Many companies deliberately use TWO different, independent DNS providers simultaneously for their most critical domains (a "secondary DNS" setup) — specifically so that if one provider has an outage, the domain remains resolvable through the other, completely independent provider. This is a direct, practical application of the "redundancy in parallel increases availability" math you learned on Day 1.

### Interview Angle
"What happens if DNS goes down?" or "How would you make DNS resilient?" → expect to mention: redundant nameservers, low TTL before planned migrations, anycast (if you want to show deeper knowledge), and possibly using multiple independent DNS providers for critical systems.

### How to teach this
> "Think of DNS like the index page at the very front of a library — if a fire alarm destroys JUST that one index page, every book in the library is still perfectly fine on its shelf, but nobody can figure out where anything is anymore. That's why libraries (and serious companies) keep BACKUP copies of the index in multiple places, and why you'd update the index well ahead of time if you were about to reorganize the shelves (a migration), rather than changing the shelves first and hoping people's memorized old directions (cached TTL) expire quickly enough."

---

## 2. LATENCY vs THROUGHPUT

### What
- **Latency**: The TIME it takes for a SINGLE request/operation to complete, start to finish — typically measured in milliseconds (ms). "How long did THIS one request take?"
- **Throughput**: The NUMBER of requests/operations a system can handle PER UNIT OF TIME — typically measured in requests per second (RPS) or queries per second (QPS). "How MANY requests can the system process in a given second?"

### Why this distinction matters (and why it's so commonly confused)
Just like Day 1's Reliability vs Availability, these two terms get used interchangeably in casual conversation, but they measure genuinely DIFFERENT things, and — critically — **improving one does NOT automatically improve the other, and sometimes improving one comes at the expense of the other.** A system design interviewer will absolutely notice if you conflate them, and being able to clearly separate the two, with a concrete example, is a strong signal of clear technical thinking.

### Background
These terms, like several others this week, trace back to general systems/operations research and computer hardware design (e.g., a CPU pipeline's "latency" for one instruction vs its "throughput" of instructions completed per clock cycle) — but they map directly and intuitively onto backend system design, especially once you think about it through the highway analogy below.

### How — The Highway Analogy (the clearest possible mental model)
Picture a highway:
- **Latency** = how long it takes ONE car to drive from the start of the highway to the end.
- **Throughput** = how many cars pass a given checkpoint per minute.

Critically: **a highway can have LOW latency (a fast speed limit, so any individual car gets through quickly) but still have LOW throughput if it only has ONE lane** (cars must travel one at a time, or close to it, limiting how many can pass per minute even though each individual one is fast). Conversely, a highway could have **somewhat higher latency per car (a lower speed limit) but much HIGHER throughput, if it has 20 lanes** running in parallel (many more cars passing the checkpoint per minute overall, even if each one individually takes a bit longer).

**This exact insight maps directly onto backend systems:**
- Adding MORE servers (horizontal scaling, Day 1/4) primarily increases **throughput** (more requests handled in parallel, system-wide) — it does NOT necessarily make any INDIVIDUAL request faster (that individual request's latency, the time for that one specific request to complete, may stay exactly the same).
- Optimizing a slow database query (e.g., adding an index — a Week 2 topic) primarily reduces **latency** for each individual query — and as a secondary effect, may ALSO improve throughput (since each request now ties up server/database resources for less time, freeing them up faster for the next request).
- A queue-based system (covered Week 3) might deliberately ADD latency (a request waits in a queue before being processed) specifically in order to achieve much higher OVERALL throughput and stability (smoothing out traffic spikes instead of letting them crash the system) — this is a genuinely important, very real trade-off you'll see again and again.

### Implementation — Measuring both in a Node.js application
```js
const express = require('express');
const app = express();

let requestCount = 0;
let windowStart = Date.now();

app.use((req, res, next) => {
  const startTime = Date.now(); // measuring THIS request's latency

  res.on('finish', () => {
    const latencyMs = Date.now() - startTime;
    console.log(`Request to ${req.path} took ${latencyMs}ms (latency)`);

    // Measuring overall THROUGHPUT - requests completed per second, system-wide
    requestCount++;
    const elapsedSeconds = (Date.now() - windowStart) / 1000;
    if (elapsedSeconds >= 1) {
      console.log(`Throughput: ${(requestCount / elapsedSeconds).toFixed(1)} requests/sec`);
      requestCount = 0;
      windowStart = Date.now();
    }
  });

  next();
});

app.get('/api/data', async (req, res) => {
  const data = await db.query('SELECT * FROM products LIMIT 10');
  res.json(data);
});

app.listen(3000);
```
Notice these are genuinely TWO separate metrics being tracked here — one measured PER REQUEST (latency), one measured ACROSS ALL requests over a time window (throughput) — and in a real production system, you'd send both of these to a monitoring/observability platform (a topic covered in full on Day 24), typically as a **latency percentile** (see below) and a **requests-per-second** graph, viewed side by side.

### A crucial related concept: Latency Percentiles (p50, p95, p99)
Simply averaging latency across all requests can be deeply misleading, because a small number of very slow ("outlier") requests can hide behind a healthy-looking average. Instead, engineers measure **percentiles**:
- **p50 (median)**: 50% of requests were faster than this value. Represents the "typical" experience.
- **p95**: 95% of requests were faster than this value (only 5% were slower). Represents a "somewhat unlucky" experience.
- **p99**: 99% of requests were faster than this value (only 1% were slower) — but at scale, 1% of a high-traffic system's requests can still represent THOUSANDS of genuinely bad experiences per day, which is why this number matters so much in production monitoring, even though it sounds small.

**Why this matters in system design discussions**: when discussing latency requirements for ANY system, a strong answer specifies WHICH percentile you're targeting — e.g., "p99 latency under 200ms" — rather than just a vague single number, because that single vague number is ambiguous about whether you mean a typical case or a near-worst case.

### Real-world example
Google famously found, through real measurement, that even very small increases in search result latency measurably reduced user engagement/searches — driving relentless internal focus on shaving milliseconds off latency. Meanwhile, a company like Kafka (Week 3 topic) is explicitly designed and marketed around extremely high THROUGHPUT (millions of messages per second across a cluster), even though any individual message might experience slightly higher latency passing through the system than, say, a direct in-memory function call would.

### Trade-offs
This is genuinely a recurring, real trade-off throughout system design: techniques that improve throughput (batching multiple requests together, queueing) often slightly INCREASE latency for any individual request (it has to wait for its batch, or wait in the queue) — and techniques that minimize latency for one request (processing it immediately, in isolation) can limit how much OVERALL throughput a system can sustain under heavy concurrent load. Knowing which one matters more for a GIVEN use case (a real-time multiplayer game cares enormously about latency; a nightly batch analytics job cares enormously about throughput, and barely at all about any single record's processing latency) is core system design judgment.

### Interview Angle
"What's the difference between latency and throughput?" — answer with the highway analogy and immediately follow with a concrete example of when you'd prioritize one over the other. Also be ready to discuss WHY percentiles (p95/p99) matter more than averages when discussing latency requirements or SLAs.

### How to teach this
> "If someone asks 'how fast is your website,' that's actually two different questions hiding inside one. 'How long does it take for ONE person to load the page' is latency. 'How many people can load the page AT THE SAME TIME without things falling apart' is throughput. A tiny food truck can serve one customer very quickly (low latency) but completely jams up if 200 people show up at once (low throughput). A huge cafeteria with 50 serving stations might take a little longer per individual person (slightly higher latency) but can comfortably handle hundreds of people simultaneously (high throughput). You usually need to think about BOTH, and they don't automatically improve together."

---

## 3. BACK-OF-THE-ENVELOPE ESTIMATION — THE NUMBERS EVERY ENGINEER SHOULD KNOW

### What
Back-of-the-envelope estimation is the practice of quickly calculating APPROXIMATE numbers (traffic volume, storage needs, bandwidth, server counts) using simple arithmetic and a small set of memorized reference numbers — to ground your system design decisions in REALITY rather than guesswork, without needing precise, exhaustive data.

### Why
Imagine designing a system and being asked "should we use a single database, or do we need sharding (Week 2)?" Without ANY estimation, you're just guessing. But if you calculate "this system will receive roughly 50,000 writes per second at peak" — that number IMMEDIATELY tells you a single traditional database almost certainly cannot keep up, and sharding or a different database type is clearly necessary. The entire REST of your design (how many servers, whether you need a cache, whether you need a CDN, whether you need sharding) flows directly from these early numbers. Skipping this step is one of the "common mistakes" flagged back on Day 1 — and now you'll learn exactly how to do it properly.

### Background
This style of rapid, approximate calculation is often called a "Fermi estimate," named after physicist Enrico Fermi, who was famous for being able to quickly approximate seemingly hard-to-know quantities (like "how many piano tuners are there in Chicago?") using a chain of reasonable assumptions and simple multiplication — without needing exact data. Big Tech companies adopted this exact style of thinking for system design interviews because it tests precisely the skill real engineers need when scoping a new system: making confident, reasonable estimates from limited information, fast.

### How — The Key Reference Numbers to Memorize

These don't need to be perfectly precise — approximate, "round" versions are exactly what's expected:

| Quantity | Approximate value |
|---|---|
| Seconds in a day | ~86,400 (often rounded to ~100,000 for quick mental math) |
| 1 KB | 1,000 bytes (or 1,024 — either is fine for estimation) |
| 1 MB | 1,000 KB |
| 1 GB | 1,000 MB |
| 1 TB | 1,000 GB |
| Average text message/tweet size | ~100-300 bytes |
| Average JSON API response | ~1-5 KB |
| Average compressed image | ~100 KB - 1 MB |
| Average compressed video (per minute) | ~10-50 MB (highly variable by quality) |
| A single server's reasonable capacity | Often estimated at a few thousand requests/sec for a simple, well-optimized API (this varies hugely, but is a usable rough starting assumption) |

### The Core Conversion You'll Use Constantly: DAU → RPS
Almost every estimation starts from "how many Daily Active Users (DAU)" and needs to become "how many Requests Per Second (RPS)." The conversion:

```
Total requests per day = DAU x average requests per user per day

Average RPS = Total requests per day / 86,400 seconds in a day
```

**Critically**: this gives you the AVERAGE RPS. Real traffic is NEVER perfectly evenly spread across 24 hours — there's always a "peak" period (e.g., evenings, lunchtime) with significantly more traffic than the average. A standard, reasonable assumption used throughout the industry and in interviews is:

```
Peak RPS = Average RPS x Peak Factor (commonly assumed as 2x to 5x average)
```

**You should ALWAYS design for peak load, not average load** — a system that handles the average just fine but falls over every single day during the predictable evening rush is not a well-designed system. This single habit (always converting average → peak before sizing anything) is one of the clearest signals of a strong candidate.

### How to teach this
> "If a national news report says 'this restaurant serves 2,400 customers a day,' that sounds calm and manageable — until you realize that almost NONE of those customers show up at 3 AM, and a huge chunk of them ALL show up between 12pm and 1pm for lunch. If you only ever planned kitchen capacity around the calm 'average' number, you'd get completely overwhelmed every single day at lunchtime. Good planning always asks: what's the busiest moment going to look like, not just the average across the whole day?"

---

## 4. THE ESTIMATION FRAMEWORK — STEP BY STEP, WITH A FULL WORKED EXAMPLE

Let's estimate the scale for a realistic system: **a photo-sharing app similar to Instagram**, and walk through every calculation an interview would expect.

### Given assumptions (the kind an interviewer might give you, or that you'd propose yourself after asking clarifying questions, per Day 1's framework)
- 100 million Daily Active Users (DAU)
- Each user views their feed an average of 10 times per day (each "feed view" = 1 read request, simplifying for this estimate)
- Each user uploads 1 photo every 10 days on average (so, 1/10 = 0.1 uploads per user per day)
- Average photo size after compression: 200 KB
- Data is retained for 5 years

### Step 1 — Calculate Read RPS (Requests Per Second)
```
Total feed-view requests per day = 100,000,000 users x 10 views/day
                                  = 1,000,000,000 requests/day (1 billion)

Average RPS = 1,000,000,000 / 86,400 seconds
            ≈ 11,574 requests/sec (let's round to ~12,000 RPS average)

Peak RPS (using a 3x peak factor, a common reasonable assumption)
            ≈ 12,000 x 3 = ~36,000 RPS at peak
```
**What this number tells us immediately**: ~36,000 RPS at peak is FAR beyond what a single server can handle (recall the rough "a few thousand RPS per server" reference number) — this single calculation, done in under a minute, already tells us we absolutely need horizontal scaling with MANY server instances behind a load balancer (Day 4), and almost certainly need a caching layer (Day 5) in front of the database for feed data, since hitting a database directly at this volume would be a serious bottleneck.

### Step 2 — Calculate Write RPS
```
Total upload requests per day = 100,000,000 users x 0.1 uploads/day
                               = 10,000,000 uploads/day

Average write RPS = 10,000,000 / 86,400 ≈ 116 writes/sec
Peak write RPS (3x factor)            ≈ 348 writes/sec
```
**What this tells us**: writes are DRAMATICALLY less frequent than reads here (~36,000 peak reads vs ~348 peak writes — roughly a 100:1 read-to-write ratio). This single insight is hugely important for system design: it tells us this system is overwhelmingly **read-heavy**, which directly justifies investing heavily in read-side optimizations like caching and read replicas (a Week 2 topic), since that's where almost all the load actually is.

### Step 3 — Calculate Storage Needs
```
Storage per day = 10,000,000 new photos/day x 200 KB/photo
                = 2,000,000,000 KB/day
                = 2,000,000 MB/day
                = 2,000 GB/day
                = ~2 TB of new photo data EVERY SINGLE DAY

Storage over 5 years = 2 TB/day x 365 days/year x 5 years
                      = 2 TB x 1,825 days
                      = ~3,650 TB
                      = ~3.65 PB (petabytes) of storage needed over 5 years
```
**What this tells us**: this is an enormous amount of data — far beyond what you'd store on traditional servers' local disks. This single number justifies the architectural decision to use **object storage** (like Amazon S3) specifically designed for storing massive volumes of large binary files cheaply and durably, rather than trying to cram this into a traditional relational database (which is built for structured, queryable data, not raw binary blobs at this volume) — you'd instead store just a small amount of METADATA (photo owner, upload time, and a pointer/URL to the actual file in object storage) in your regular database.

### Step 4 — Calculate Bandwidth
```
Upload bandwidth = 10,000,000 photos/day x 200 KB
                  = 2,000,000,000 KB/day, spread across 86,400 seconds
                  = ~23,150 KB/sec ≈ ~23 MB/sec average upload bandwidth needed

(For reads, you'd similarly multiply feed-view requests by however much
photo data is actually being VIEWED per feed load, which would be
significantly higher than the upload bandwidth, given the 100:1 read:write
ratio calculated above - reinforcing yet again that this is a read-heavy
system, and reinforcing why a CDN, Day 5, is essential here specifically
to avoid all that READ bandwidth hitting your origin servers directly)
```

### Putting it together — what this estimation JUSTIFIES in the design
Notice how every single architectural decision below is now backed by an actual number, not a guess:
- **Load balancer + many horizontally-scaled app servers** (Day 4) — justified by ~36,000 peak RPS, far beyond one server.
- **Heavy caching layer for feed reads** (Day 5) — justified by the 100:1 read-to-write ratio.
- **Object storage (S3) for photos, not a traditional database** — justified by ~3.65 PB over 5 years.
- **CDN for serving photos** (Day 5) — justified by the read-heavy bandwidth needs, and by knowing users are geographically distributed globally.
- **Read replicas for the metadata database** (a Week 2 topic) — justified, again, by the read-heavy ratio.

This is EXACTLY the kind of reasoning chain ("here's the number, and here's the specific design decision it justifies") that a strong system design interview answer demonstrates throughout.

---

## 5. COMMON ESTIMATION PITFALLS

1. **Forgetting to convert average to peak.** Always apply a peak factor (commonly 2x-5x) — designing only for the average is a classic, easily-avoided mistake.
2. **Spending too much time on precision.** The goal is APPROXIMATE, fast numbers, not perfect accuracy — rounding 11,574 to "about 12,000" or even "about 10,000" is completely fine and expected; interviewers are testing your reasoning process, not your arithmetic precision.
3. **Forgetting units halfway through a calculation.** Mixing up KB and MB, or seconds and days, is the most common actual ERROR (as opposed to reasoning mistake) people make under interview pressure — explicitly writing out units at every step (as shown in the worked example above) prevents this.
4. **Not stating assumptions out loud.** If given DAU but not "requests per user," you must EXPLICITLY state your assumption ("let's assume each user makes about 10 requests per day") rather than silently picking a number — interviewers want to see and validate your assumptions, not just your final answer.
5. **Treating the estimation as a one-time, throwaway step.** As shown in Step 4 above, the BEST candidates continuously refer back to these numbers throughout the rest of the design discussion ("given our ~36,000 peak RPS, I'd want at least N server instances behind the load balancer, each handling roughly...") rather than calculating once and never mentioning the numbers again.

### Interview Angle
Estimation is almost always explicitly expected as **Step 2 of the 7-step framework** from Day 1. Interviewers are often more interested in HOW you reason through the calculation (do you ask the right clarifying questions, do you convert average to peak, do you connect the resulting number back to an actual design decision) than whether your final number is exactly "correct" — there often isn't one single correct number anyway, since it depends on the assumptions you state.

---

## 6. DAY 6 CHEAT SHEET

```
DNS RESILIENCE
  DNS is a single point of failure in the request chain (Day 1 series-availability math)
  Mitigations: redundant nameservers, lower TTL before migrations, anycast DNS,
  sometimes even two independent DNS providers for critical domains

LATENCY vs THROUGHPUT
  Latency    = time for ONE request (ms) - "how long did this take?"
  Throughput = requests handled per unit time (RPS) - "how many at once?"
  Highway analogy: fast lane = low latency; many lanes = high throughput
  These don't automatically improve together - batching/queueing often
  trades a bit of latency for a lot more throughput
  Use PERCENTILES (p50/p95/p99) for latency, not plain averages -
  p99 hides the "1% of users having a bad time" that still matters at scale

ESTIMATION REFERENCE NUMBERS
  Seconds/day ~86,400 (round to ~100,000 for quick math)
  1 KB=1000 bytes, 1 MB=1000 KB, 1 GB=1000 MB, 1 TB=1000 GB
  Always convert AVERAGE -> PEAK using a 2x-5x peak factor

ESTIMATION FRAMEWORK
  1. DAU -> total requests/day (DAU x actions/user/day)
  2. Total requests/day -> average RPS (divide by 86,400)
  3. Average RPS -> peak RPS (multiply by peak factor)
  4. Storage = (new records/day) x (size/record) x (retention period)
  5. Bandwidth = (records/day) x (size/record), spread across seconds/day
  6. CONNECT every number to an actual design decision -
     don't just calculate and move on

PITFALLS TO AVOID
  Forgetting peak factor | Over-precision | Unit mix-ups |
  Silent unstated assumptions | Calculating once and never reusing the numbers
```

---

## WEEK 1 COMPLETE — WHAT YOU NOW KNOW

Across Days 1-6, you've built the entire FOUNDATION layer of system design knowledge:
- **Day 1**: How to approach any problem (the 7-step framework), and the 4 pillars (Scalability, Reliability, Availability, Maintainability).
- **Day 2**: The actual plumbing — client-server, DNS, HTTP/HTTPS, TCP/UDP.
- **Day 3**: API design paradigms — REST, GraphQL, gRPC, WebSockets.
- **Day 4**: Scaling and load balancing — stateless design, algorithms, health checks.
- **Day 5**: Proxies, CDNs, and caching — including hand-building an LRU cache.
- **Day 6**: Estimation — the math that justifies every design decision you'll ever make.

**Day 7 is your Week 1 capstone**: designing a complete URL Shortener from scratch, applying EVERY single concept above, end-to-end — requirements, estimation, API design, high-level architecture, caching strategy, and a working Node.js implementation. This is also one of the single most common actual system design interview questions asked in the real world, so it doubles as direct interview practice.

**Say "Day 7" whenever you're ready.**
