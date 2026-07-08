# Module 3 — Networking Basics for System Design

> **Masterclass:** System Design Masterclass (30 Modules)
> **Level:** Beginner
> **Audience:** Node.js backend developers, SDE‑2 / Senior Backend interview candidates, engineers transitioning into architecture roles
> **Prerequisite:** Module 1 — Introduction to System Design, Module 2 — Scalability Fundamentals

---

## 1. Introduction

In Module 2 we drew load balancers, multiple servers, and Redis stores as boxes connected by arrows — but we never asked the question those diagrams silently assume an answer to: **how does a request actually travel from a user's phone to any of those boxes at all?**

Every system design decision from here forward — load balancing (Module 8), CDNs (Module 10), API gateways (Module 9), even security (Module 20) — is built on top of a small set of networking primitives: IP addresses, DNS, ports, routers, and firewalls. If these are fuzzy, every later module will feel like memorized configuration rather than reasoned design. This module makes them concrete, from first principles, without assuming any prior networking coursework.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

1. Explain what an **IP address** is and why both IPv4 and IPv6 exist.
2. Explain **DNS resolution** step by step, including why caching at multiple levels matters.
3. Explain what a **port** is and why a single machine can run many services simultaneously.
4. Distinguish the roles of a **router**, **switch**, **NAT device**, and **firewall** in a network path.
5. Explain **packet** transmission and why large messages get broken into pieces.
6. Reason about **bandwidth vs. latency** as two independent network properties.
7. Diagram the full network path a request takes from a browser to a cloud-hosted backend.
8. Identify common networking misconfigurations that cause real production outages (e.g., exposed database ports, missing firewall rules, DNS TTL issues).

---

## 3. Why This Concept Exists

Every diagram we've drawn so far has an arrow labeled simply "request" going from a client to a server. That arrow is doing an enormous amount of hidden work: resolving a human-readable name into a machine-routable address, traversing potentially dozens of physical network hops across the public internet, arriving at the correct *port* on the correct *machine*, and passing through firewalls that decide whether the request is even allowed to proceed.

Networking, as a discipline within system design, exists because **distributed systems are fundamentally defined by machines that must communicate over networks they do not control** — the public internet, with its variable latency, occasional packet loss, and adversarial actors. Every reliability guarantee, every latency budget, and every security boundary in this course ultimately rests on networking fundamentals. You cannot reason correctly about a CDN's benefit (Module 10) without understanding that physical distance causes real, unavoidable latency — which is exactly what this module establishes.

---

## 4. Problem Statement

> Our horizontally-scaled blog platform (Module 2) is deployed across 5 servers in a single cloud region. Users worldwide report inconsistent load times — some see the site load in 50ms, others in 800ms. Additionally, a security audit flagged that the PostgreSQL database port is reachable from the public internet. Explain, using core networking concepts, why these two problems occur, and how the network topology should be structured to prevent them.

---

## 5. Real-World Analogy

**The internet is the postal system, and networking primitives are its addressing and routing conventions.**

- An **IP address** is like a street address — a unique identifier for where a package (data) should go.
- **DNS** is the phone book / contacts list that converts a name ("Grandma's house") into an actual street address, because humans remember names, not addresses.
- A **port** is the apartment number within a building — the street address gets you to the right building (machine), the port gets your package to the right apartment (service/process) inside it.
- A **router** is a postal sorting facility — it looks at the destination address and decides which direction to forward the package, without needing to understand the package's contents.
- A **switch** is the internal mail-sorting within one office building — routing between machines on the *same* local network.
- **NAT (Network Address Translation)** is like a company mailroom that receives all mail addressed to "Acme Corp, Suite 500" and internally redistributes it to individual employees' desks — the outside world only ever sees one company address, not each employee's desk location.
- A **firewall** is the security guard at the building's front desk, checking a list of who's allowed in and turning away anyone not on it.
- **Bandwidth** is how many packages the delivery truck can carry per trip; **latency** is how long the truck takes to make one trip, regardless of how full it is.

---

## 6. Technical Definition

**IP Address:** A unique numerical identifier assigned to a device on a network, enabling routers to determine where to deliver data. IPv4 addresses are 32-bit (e.g., `203.0.113.10`); IPv6 addresses are 128-bit, created because IPv4's ~4.3 billion address space is insufficient for the number of internet-connected devices today.

**DNS (Domain Name System):** A distributed, hierarchical naming system that translates human-readable domain names (`myblog.com`) into IP addresses.

**Port:** A 16-bit number (0–65535) that identifies a specific process or service on a machine, allowing multiple network services to run on the same IP address simultaneously.

**Router:** A network device that forwards data packets between different networks based on destination IP addresses.

**Switch:** A network device that forwards data between devices *within* the same local network, typically using MAC addresses.

**NAT (Network Address Translation):** A method of remapping one IP address space into another, commonly used to allow multiple devices on a private network to share a single public IP address.

**Firewall:** A network security system that monitors and controls incoming and outgoing traffic based on predetermined security rules.

**Packet:** A formatted unit of data transmitted over a network, containing both a payload and header metadata (source/destination addresses, sequencing info).

---

## 7. Core Terminology

| Term | Precise Definition | One-line Intuition |
|---|---|---|
| **IP Address** | Numeric network identifier | "Street address" |
| **DNS** | Name → IP translation system | "Phone book" |
| **Port** | Process identifier on a machine | "Apartment number" |
| **Router** | Forwards traffic between networks | "Postal sorting facility" |
| **Switch** | Forwards traffic within a network | "Internal office mail room" |
| **NAT** | Maps private IPs to a shared public IP | "Company mailroom" |
| **Firewall** | Filters traffic by rule | "Security guard" |
| **Packet** | Unit of transmitted data | "One sealed envelope" |
| **Bandwidth** | Data volume per unit time | "Truck size" |
| **Latency (network)** | Time for a packet to travel one-way (or round-trip) | "Trip duration" |

### Bandwidth vs. Latency — why more bandwidth doesn't fix a slow single request

A common misconception: "our API is slow, let's upgrade to a bigger network pipe (more bandwidth)." This only helps if the bottleneck is *volume* of concurrent data, not the *travel time* of any single packet.

**Analogy check:** if one truck takes 3 hours to drive from New York to Los Angeles, buying a *bigger* truck (more bandwidth) does not make that one trip faster — it only lets you carry more cargo *per trip*. To make the trip itself faster, you need a *closer* origin point (this is precisely what a CDN, Module 10, provides) or a fundamentally faster transport mechanism.

This distinction directly explains the Section 4 problem statement: users far from our single cloud region experience high **latency** (the physical distance the packets must travel), which no amount of additional **bandwidth** at our origin server can fix.

---

## 8. Internal Working

### How DNS resolution actually works, step by step

When a browser requests `https://myblog.com`, here is the real resolution chain (simplified but accurate):

1. **Browser cache check** — has this domain been resolved recently? If yes and within TTL, use the cached IP immediately (skip to step 6).
2. **OS cache check** — same check at the operating system level.
3. **Recursive resolver** (typically your ISP's or a public one like `8.8.8.8`) is queried if no cache hit.
4. The recursive resolver queries a **root DNS server**, which points to the correct **TLD server** (e.g., for `.com`).
5. The TLD server points to the **authoritative name server** for `myblog.com`, which finally returns the actual IP address.
6. The browser now has the IP address and can open a **TCP connection** to it (and a TLS handshake, if HTTPS — full detail in Module 4).

**Why caching at every level matters:** without caching, *every single request* to any website would require this entire multi-hop lookup chain, adding significant latency to every page load. **TTL (Time To Live)** on a DNS record controls how long each cache layer is allowed to trust a resolved IP before re-checking — this is a real trade-off: low TTL means faster updates when you change infrastructure (e.g., during a failover) but more DNS lookup overhead; high TTL means better caching efficiency but slower propagation of changes.

### How NAT solves address scarcity

Your home has one public IP address from your ISP, but multiple devices (laptop, phone, smart TV) are all "online" simultaneously. Your home router performs NAT: it maintains an internal table mapping each device's private IP (e.g., `192.168.1.5`) plus a chosen port to the single public IP, so responses from the internet know which internal device to route back to. This is the exact same conceptual mechanism used inside cloud VPCs (Virtual Private Clouds) to let backend servers with only *private* IPs still make *outbound* internet calls (e.g., to a third-party API) without being directly reachable *from* the internet — a critical security pattern referenced again in Section 19.

---

## 9. Request Lifecycle

### Mermaid Sequence Diagram — Full Network Path, DNS Through Response

```mermaid
sequenceDiagram
    participant B as Browser
    participant OS as OS DNS Cache
    participant R as Recursive Resolver
    participant Root as Root DNS Server
    participant TLD as TLD Server (.com)
    participant Auth as Authoritative NS (myblog.com)
    participant LB as Load Balancer (Public IP)
    participant App as App Server (Private IP)

    B->>OS: Resolve myblog.com?
    OS-->>B: Cache miss
    B->>R: Query myblog.com
    R->>Root: Where is .com?
    Root-->>R: TLD server address
    R->>TLD: Where is myblog.com?
    TLD-->>R: Authoritative NS address
    R->>Auth: Resolve myblog.com
    Auth-->>R: IP: 203.0.113.10
    R-->>B: IP: 203.0.113.10 (cached with TTL)
    B->>LB: TCP connect + HTTPS GET (to public IP)
    LB->>App: Forward request (to private IP, inside VPC)
    App-->>LB: Response
    LB-->>B: Response
```

**Step-by-step explanation:** notice that the **app server's IP is never directly contacted by the browser** — only the load balancer's public IP is. This single detail is the technical backbone of the network security posture we established informally in Modules 1 and 2, and now have the vocabulary to state precisely: **the app server has no public IP, and NAT/private networking ensures it can still reach the internet outbound (e.g., for third-party API calls) without being reachable inbound.**

---

## 10. Architecture Overview

```mermaid
flowchart TB
    subgraph Public Internet
        User[User Browser]
        DNS[DNS Resolvers]
    end
    subgraph Cloud VPC - Public Subnet
        LB[Load Balancer\nPublic IP: 203.0.113.10]
    end
    subgraph Cloud VPC - Private Subnet
        App1[App Server 1\nPrivate IP: 10.0.1.5]
        App2[App Server 2\nPrivate IP: 10.0.1.6]
        DB[(PostgreSQL\nPrivate IP: 10.0.2.10)]
    end
    User -->|1. DNS lookup| DNS
    DNS -->|2. returns Public IP| User
    User -->|3. HTTPS request| LB
    LB -->|4. forwards internally| App1
    LB -->|4. forwards internally| App2
    App1 & App2 -->|5. private network only| DB
```

**Why this satisfies the Section 4 security concern:** the database's private IP (`10.0.2.10`) is **not routable from the public internet at all** — it exists only within the VPC's private address space. Even if someone knows this IP, there is no network path from outside the VPC to reach it, which is a fundamentally stronger guarantee than a firewall rule alone (defense in depth — belt *and* suspenders).

---

## 11. Capacity Estimation

Networking capacity estimation centers on **bandwidth**, not request count directly.

**Scenario:** Our blog serves 500 req/s at peak, and the average response payload (JSON + headers) is 15 KB.

**Step 1 — Total outbound data per second:**
```
500 req/s × 15 KB = 7,500 KB/s = 7.5 MB/s
```

**Step 2 — Convert to bits per second (networking is typically specified in bits, not bytes):**
```
7.5 MB/s × 8 = 60 Mbps (megabits per second)
```

**Step 3 — Compare against provisioned capacity:** if our load balancer's network interface is provisioned for 1 Gbps, we are using only 6% of available bandwidth at peak — meaning **bandwidth is not our current bottleneck**, and our earlier diagnosis (Section 4: latency due to geographic distance, not bandwidth saturation) is numerically confirmed. This is exactly the kind of number that should accompany any claim about network bottlenecks in an interview — "bandwidth might be the issue" is a guess; "we're using 6% of 1 Gbps, so it isn't" is an answer.

---

## 12. High-Level Design (HLD)

Given the Section 4 problem (global users experiencing inconsistent latency due to a single-region deployment), the HLD-level insight this module equips you to state — *without yet knowing the full solution* — is:

```mermaid
flowchart LR
    subgraph Single Region Today
        UserFar[User in Asia] -->|high latency: long physical distance| Origin[Single Region: us-east-1]
        UserNear[User in US East] -->|low latency: short physical distance| Origin
    end
```

**The correct HLD-level statement (without yet designing the fix):** "This latency disparity is a direct, physical consequence of network distance, and cannot be solved by scaling compute (Module 2) or optimizing code — it requires either **multi-region deployment** or a **CDN** (Module 10) to place content/compute physically closer to distant users." Recognizing *which category* of fix applies, even before learning the fix's mechanics, is the actual skill being assessed here and in interviews.

---

## 13. Low-Level Design (LLD)

### Inspecting the real network path (a skill, not just theory)

```bash
# See the DNS resolution and IP address for a domain
dig myblog.com +short

# See how many network hops (routers) a request travels through
traceroute myblog.com   # Linux/Mac
tracert myblog.com      # Windows

# See the actual open ports on a server you control
netstat -tuln           # or: ss -tuln (modern Linux)
```

### Example `traceroute` output (illustrative)

```
1  10.0.0.1        1.2 ms   (home router)
2  100.65.12.1     8.4 ms   (ISP router)
3  72.14.238.234   15.1 ms  (ISP backbone)
4  108.170.242.1   45.3 ms  (transit provider)
5  203.0.113.10    52.7 ms  (destination load balancer)
```

**Reading this correctly:** each hop adds latency. This is a direct, empirical demonstration of why physical network topology — not just server-side code — determines a meaningful portion of total response time, reinforcing the Section 11 conclusion.

---

## 14. ASCII Diagrams

```
PORT MULTIPLEXING ON ONE MACHINE (one IP, many services)

        IP: 203.0.113.10
               │
      ┌────────┼────────┬─────────┐
      ▼        ▼         ▼         ▼
  Port 80   Port 443   Port 22   Port 5432
  (HTTP)    (HTTPS)     (SSH)   (PostgreSQL)
     │         │          │         │
   Web App   Web App   Remote    Database
  (redirect)          Admin Access  Server
```

```
NAT TRANSLATION TABLE (conceptual)

  Private Network              NAT Device            Public Internet
  10.0.1.5:51000  ──────┐
  10.0.1.6:51002  ──────┼──▶ [Translation Table] ──▶ 203.0.113.10:*
  10.0.1.7:51004  ──────┘         maps each
                              private conn to a
                              unique public port
```

---

## 15. Mermaid Flowcharts

### Decision Flow: Diagnosing a "Slow API" Complaint

```mermaid
flowchart TD
    A[User reports slow API] --> B{Is latency high for ALL users or SOME?}
    B -->|All users, all regions| C[Likely app/DB bottleneck - not networking]
    B -->|Only users far from your region| D[Likely network/geographic latency]
    D --> E{Is bandwidth actually saturated?}
    E -->|No, well under capacity| F[Confirms: distance is the cause, not bandwidth]
    F --> G[Consider CDN or multi-region deployment - Module 10]
    E -->|Yes, near capacity| H[Bandwidth upgrade or payload size reduction needed]
```

---

## 16. Mermaid Sequence Diagrams

### DNS Failover Scenario — Why TTL Matters

```mermaid
sequenceDiagram
    participant U as User
    participant Cache as DNS Cache (TTL=3600s)
    participant Auth as Authoritative DNS

    U->>Cache: Resolve myblog.com
    Cache-->>U: IP: 203.0.113.10 (cached, 1hr TTL)
    Note over Auth: Ops team changes IP to 203.0.113.99\ndue to server migration
    U->>Cache: Resolve myblog.com again (30 min later)
    Cache-->>U: Still returns OLD IP: 203.0.113.10 (TTL not expired!)
    Note over U: User's requests fail or hit decommissioned server\nuntil the 1-hour TTL fully expires
```

**Why this matters operationally:** teams doing planned migrations often **lower the DNS TTL in advance** (e.g., from 3600s to 60s) so that when the actual IP cutover happens, clients pick up the change within a minute instead of up to an hour. This is a genuinely common, practical pre-migration checklist item.

---

## 17. Component Diagrams

```mermaid
flowchart LR
    subgraph Network Edge
        FW[Firewall / Security Group]
        LB[Load Balancer]
    end
    subgraph Application Network Tier
        Router1[Internal Router/Gateway]
        App[App Servers]
    end
    subgraph Data Network Tier
        DBFW[Database Security Group\nallows ONLY App Tier IPs]
        DB[(Database)]
    end

    Internet --> FW --> LB --> Router1 --> App --> DBFW --> DB
```

**Why the database has its *own* firewall (security group), separate from the network edge firewall:** this is **defense in depth** — even if the edge firewall is misconfigured, the database's own rule ("only accept connections from App Tier IP ranges") provides a second, independent layer of protection. Relying on a single firewall layer anywhere in a production system is itself an anti-pattern (Section 29).

---

## 18. Deployment Diagrams

```mermaid
flowchart TB
    subgraph VPC["Cloud VPC (10.0.0.0/16)"]
        subgraph PublicSubnet["Public Subnet (10.0.0.0/24)"]
            LB[Load Balancer]
            NAT_GW[NAT Gateway]
        end
        subgraph PrivateSubnet["Private Subnet (10.0.1.0/24)"]
            App1[App Server 1]
            App2[App Server 2]
        end
        subgraph DataSubnet["Data Subnet (10.0.2.0/24)"]
            DB[(PostgreSQL)]
        end
    end
    Internet <--> LB
    LB --> App1
    LB --> App2
    App1 -.outbound only, via NAT.-> NAT_GW
    App2 -.outbound only, via NAT.-> NAT_GW
    NAT_GW <--> Internet
    App1 & App2 --> DB
```

**Deployment-level explanation:** the **Private Subnet** has no direct internet route — app servers can reach the internet *outbound* (e.g., to call a third-party payment API) only through the **NAT Gateway**, but nothing on the internet can initiate a connection *into* the private subnet. The **Data Subnet** is even more restricted — it typically has no NAT gateway route at all, since a database rarely needs outbound internet access.

---

## 19. Network Diagrams

```
                          INTERNET
                              │
                    ┌─────────▼─────────┐
                    │   Security Group:  │
                    │  Allow 443 from    │
                    │  0.0.0.0/0 (any)   │
                    └─────────┬─────────┘
                    ┌─────────▼─────────┐
                    │   Load Balancer    │  Public Subnet
                    │  10.0.0.5 (public) │
                    └─────────┬─────────┘
                    ┌─────────▼─────────┐
                    │  Security Group:   │
                    │ Allow 3000 from    │
                    │ LB's SG only       │
                    └─────────┬─────────┘
                    ┌─────────▼─────────┐
                    │   App Servers      │  Private Subnet
                    │ 10.0.1.x (private) │
                    └─────────┬─────────┘
                    ┌─────────▼─────────┐
                    │  Security Group:   │
                    │ Allow 5432 from    │
                    │ App Tier SG ONLY   │
                    └─────────┬─────────┘
                    ┌─────────▼─────────┐
                    │    PostgreSQL       │  Data Subnet
                    │ 10.0.2.x (private)  │  ← NOT reachable from internet
                    └────────────────────┘
```

**This diagram directly resolves the Section 4 audit finding.** The fix isn't "add a firewall rule" in isolation — it's **placing the database in a private subnet with no public IP and no internet gateway route at all**, so that even a misconfigured security group rule cannot expose it, because there is *no network path* to attempt the connection over in the first place. This is a stronger guarantee than firewall rules alone, and is exactly what "security groups + subnet architecture" means in real cloud production systems.

---

## 20. Database Design

Networking's influence on database design at this stage is primarily about **connectivity topology**, not schema:

- The database should reside in a **dedicated, isolated subnet** (Section 18/19) with no public route.
- **Connection strings** used by app servers should reference the database's **private DNS name** (e.g., `db.internal.myblog.local`) rather than a raw IP — private DNS allows the underlying IP to change (e.g., during a failover to a replica, Module 15) without requiring every app server's configuration to be updated.

```
# BAD: hardcoded private IP — breaks silently on any infrastructure change
DATABASE_URL=postgres://user:pass@10.0.2.10:5432/blog

# GOOD: private DNS name — infrastructure can change underneath it
DATABASE_URL=postgres://user:pass@db.internal.myblog.local:5432/blog
```

---

## 21. API Design

Networking rarely changes the *shape* of an API contract, but it introduces two real, practical considerations:

1. **Timeouts must account for network variability.** A client-side timeout of 200ms is unrealistic for users on high-latency mobile networks in distant regions — this is a direct consequence of Section 11's latency math, and should be set based on measured p99 network RTT (round-trip time) for your actual user base, not an arbitrary guess.
2. **Idempotency matters more under unreliable networks.** If a client's request times out on the network but the server actually processed it, a naive retry can cause duplicate actions (e.g., double-charging a payment). Designing `POST` endpoints to accept an idempotency key is a direct, practical response to the fact that networks are inherently unreliable — you cannot assume "no response" means "nothing happened."

---

## 22. Scalability Considerations

| Consideration | Why Networking Matters |
|---|---|
| Geographic user distribution | Determines whether multi-region/CDN (Module 10) is needed, independent of compute scaling (Module 2) |
| DNS TTL strategy | Low TTL enables fast failover/scaling changes; high TTL reduces DNS query load |
| NAT Gateway throughput | NAT gateways have their own bandwidth/connection limits and can become a hidden bottleneck at scale |
| Load balancer network interface capacity | Must be provisioned ahead of raw compute scaling, or it becomes the new bottleneck (Section 27) |

---

## 23. Reliability & Fault Tolerance

- A **single NAT Gateway** is itself a potential single point of failure for all outbound traffic from a private subnet — production deployments typically provision one NAT Gateway **per availability zone**, not one for the entire region.
- **DNS itself can fail or be slow** — this is why production systems often set conservative DNS caching (balancing TTL) and monitor DNS resolution time as a first-class metric (Module 19).
- Multiple, redundant paths at the network level (multiple availability zones, redundant routers) are the network-layer expression of the same "no single point of failure" principle established in Module 1 for compute.

---

## 24. Security Considerations

This module's central security lesson, worth restating explicitly:

> **Network-layer isolation (private subnets, no public IP) is a stronger security guarantee than firewall rules alone, because it removes the network path entirely rather than relying on a rule to correctly block it.**

Additional considerations:
- **Firewalls/security groups should follow least-privilege**: only open the exact ports needed, from the exact source IP ranges needed (e.g., database security group allows only the app tier's security group, never `0.0.0.0/0`).
- **NAT does not equal security** — NAT was invented for address scarcity, not protection; conflating "my server has a private IP behind NAT" with "my server is secure" is a common and dangerous misunderstanding. Private IPs still need proper firewall rules; they simply aren't *directly* internet-routable by default.

---

## 25. Performance Optimization

- **Reduce DNS lookups** for third-party dependencies where possible (each unique domain requires its own DNS resolution).
- **Use HTTP keep-alive / connection reuse** so repeated requests to the same server don't pay the full TCP + TLS handshake cost every time (deepened in Module 4).
- **Right-size DNS TTL**: too low increases DNS query volume and resolver load; too high slows down legitimate infrastructure changes (Section 16).
- **Co-locate dependent services** (app server and database) in the same region/availability zone where possible, to minimize inter-service network latency — cross-region calls between your *own* services are a frequently overlooked source of avoidable latency.

---

## 26. Monitoring & Observability

Network-specific metrics worth tracking, beyond the application metrics from Module 1:

- **DNS resolution time** (should be milliseconds; spikes indicate resolver issues)
- **TCP connection establishment time** (separate from total request time — isolates network setup cost)
- **Packet loss / retransmission rate** (a rising trend often precedes visible latency degradation)
- **NAT Gateway connection count** (approaching limits causes silent connection failures)
- **Per-region latency breakdown** — aggregate latency numbers hide the exact problem described in Section 4; you must segment by user geography to see it.

---

## 27. Common Bottlenecks

| Bottleneck | Symptom | Root Cause |
|---|---|---|
| Geographic distance | High latency for distant users only | No CDN/multi-region presence (Module 10) |
| DNS TTL misconfiguration | Stale routing after infrastructure change | TTL too high relative to change frequency |
| NAT Gateway limits | Intermittent outbound connection failures at scale | Single NAT gateway under-provisioned for connection count |
| Load balancer NIC saturation | Degraded performance despite low app CPU | Network interface bandwidth limit reached before compute limit |
| Overly broad firewall rules | Security incidents, unauthorized access | `0.0.0.0/0` rules where specific IP ranges should be used |

---

## 28. Trade-off Analysis

> "I chose to place the database in a fully isolated private subnet with no internet gateway route, rather than relying solely on security group rules, optimizing for **defense-in-depth security**, at the cost of **slightly more complex network topology to manage and provision**, which is acceptable because database breaches are high-severity, low-tolerance events where an extra layer of protection is worth the added setup complexity."

> "I chose a **low DNS TTL (60s)** during our migration window rather than the standard 3600s, optimizing for **fast, safe failover**, at the cost of **increased DNS query volume and resolver load** during that window, which is acceptable because it's a temporary, planned trade-off — I reverted the TTL to 3600s once the migration was confirmed stable."

---

## 29. Anti-patterns & Common Mistakes

1. **Exposing database or admin ports directly to the internet** (`0.0.0.0/0` on port 5432/3306/22) — the single most common real-world networking security misconfiguration, and the exact issue flagged in this module's Section 4 problem statement.
2. **Confusing NAT with security** — assuming a private IP is inherently protected without proper firewall rules (Section 24).
3. **Hardcoding IP addresses** instead of using private DNS names, causing silent breakage on any infrastructure change (Section 20).
4. **Ignoring DNS TTL during planned migrations**, causing a long, confusing tail of requests still hitting decommissioned infrastructure.
5. **Diagnosing "slow API" as a compute problem** without first checking whether the issue is geographically localized — a classic case of solving the wrong layer of the stack (Section 12/15).
6. **A single NAT Gateway for an entire multi-AZ deployment** — creates a hidden single point of failure for all outbound traffic.

---

## 30. Production Best Practices

- Place databases and other sensitive data stores in **private subnets with no internet gateway route** — not just firewall-protected public subnets.
- Use **private DNS names**, not hardcoded IPs, for all internal service-to-service communication.
- **Lower DNS TTL in advance** of any planned infrastructure migration or failover event; restore it afterward.
- Provision **one NAT Gateway per availability zone**, not a single shared one, for both reliability and reduced cross-AZ data transfer costs.
- Monitor network metrics (DNS resolution time, TCP connect time, packet loss) as **first-class signals**, not an afterthought to application metrics.
- Apply **least-privilege firewall rules** everywhere — never default to `0.0.0.0/0` for anything beyond a public load balancer's HTTPS port.

---

## 31. Real-World Examples

- **The 2017 GitLab outage** and numerous other well-documented incidents trace back partly to overly permissive network access controls combined with insufficient isolation between production data stores and broader infrastructure — a real-world instance of the exact anti-pattern in Section 29.1.
- **Cloudflare** and other CDN/edge providers exist entirely because of the physics established in this module — no amount of origin server optimization changes the speed of light across a transoceanic cable; only physical proximity (edge locations) can reduce that specific latency component, which is the entire premise of Module 10.
- **AWS, GCP, and Azure's VPC models** all directly implement the public-subnet/private-subnet/NAT-gateway pattern described in Section 18 as their default recommended production architecture — this is not a theoretical construct invented for this course, it is the literal, standard blueprint used across virtually all serious cloud deployments today.

---

## 32. Node.js Implementation Examples

### Setting sensible network timeouts (a direct, practical response to network unreliability)

```javascript
const axios = require('axios');

// BAD: no timeout — a hung network call can block indefinitely
async function fetchThirdPartyBad() {
  const res = await axios.get('https://partner-api.example.com/data');
  return res.data;
}

// GOOD: explicit timeout, based on measured p99 latency + margin
async function fetchThirdPartyGood() {
  try {
    const res = await axios.get('https://partner-api.example.com/data', {
      timeout: 3000, // 3s — chosen from measured p99 RTT, not guessed
    });
    return res.data;
  } catch (err) {
    if (err.code === 'ECONNABORTED') {
      console.error('Third-party call timed out — network or partner latency issue');
    }
    throw err;
  }
}
```

### Resolving DNS manually to inspect resolution time (diagnostic tool)

```javascript
const dns = require('dns').promises;

async function measureDnsResolution(hostname) {
  const start = Date.now();
  const addresses = await dns.resolve4(hostname);
  console.log(`Resolved ${hostname} to ${addresses} in ${Date.now() - start}ms`);
  return addresses;
}

measureDnsResolution('myblog.com');
```

---

## 33. Interview Questions

### Easy
1. What is the difference between an IP address and a port?
2. Explain, in your own words, what DNS does and why it's necessary.
3. What is the role of a firewall in a network path?
4. Why does IPv6 exist if IPv4 already provides billions of addresses?
5. What's the difference between a router and a switch?
6. Why shouldn't a production database have a public IP address?

### Medium
7. Explain the full DNS resolution chain from browser cache to authoritative name server.
8. What is NAT, and why is it sometimes mistakenly believed to provide security on its own?
9. Why might lowering DNS TTL before a migration be a good idea, and what's the trade-off?
10. A user in India reports 10x higher latency than a user in the same city as your servers. Using only networking concepts (not caching/CDN specifics), explain why.
11. What's the difference between bandwidth and latency, and can you improve one without improving the other?
12. Why is it good practice to use private DNS names instead of hardcoded IPs for internal service communication?

### Hard
13. Design the subnet and security group topology for a 3-tier application (web, app, database) with a strong security posture, and justify each isolation boundary.
14. Explain why a single NAT Gateway across multiple availability zones is a reliability risk, using the SPOF concept from Module 1.
15. A `traceroute` shows 12 hops with a large latency jump between hop 6 and hop 7. What are two possible explanations, and how would you investigate further?
16. Design an idempotency strategy for a payment API endpoint, explaining specifically how unreliable networks motivate this design (not just "duplicate clicks").
17. Explain, using both networking and caching concepts, why a CDN reduces latency in a way that simply adding more app servers in your existing single region cannot.

---

## 34. Scenario-Based Design Questions

1. **Scenario:** A security audit finds your database is reachable from the public internet despite a "deny all" firewall rule you believe is correctly configured. What network-level design change would you propose as a stronger, defense-in-depth fix?
2. **Scenario:** During a planned server migration, users experience an hour of intermittent failures even after the new servers are confirmed healthy. Diagnose using DNS concepts.
3. **Scenario:** Your app calls a third-party payment API with no explicit timeout configured, and during a partner outage, your own API becomes unresponsive. Explain the mechanism and the fix.
4. **Scenario:** You're asked to reduce latency for users in Australia while your infrastructure is entirely in `us-east-1`, without yet implementing a CDN or multi-region deployment (Module 10 territory) — what, if anything, can be improved purely at the networking/config level today?
5. **Scenario:** Your NAT Gateway is reporting connection errors under peak load, but your app servers show low CPU usage. Explain the likely cause and fix.
6. **Scenario:** A teammate proposes hardcoding your database's IP address into every app server's config "for simplicity." Push back with a concrete failure scenario this would cause.
7. **Scenario:** Two services in the same region but different availability zones communicate heavily and you're seeing unexpectedly high latency and cost. What networking-level design question would you ask first?
8. **Scenario:** Your load balancer's network interface metrics show near-100% utilization while CPU is at 30%. What does this tell you, and what's your next step?
9. **Scenario:** An engineer argues "since our servers are behind NAT, we don't need a firewall." Explain, precisely, why this reasoning is flawed.
10. **Scenario:** You need to support a mobile app used in regions with poor cellular connectivity. How does this specifically change your approach to API timeouts and retry design?

---

## 35. Hands-on Exercises

1. Run `dig` (or `nslookup`) against three different domains and note the TTL values returned; explain what each TTL implies about how often that domain's operators expect their IP to change.
2. Run `traceroute`/`tracert` to a website hosted on a different continent from you, and to one hosted in your own country; compare hop counts and total latency.
3. Use `netstat -tuln` (or `ss -tuln`) on your own machine to list all currently open/listening ports, and identify what process/service owns each one.
4. Design (on paper or in Mermaid) a 3-subnet VPC topology (public, private-app, private-data) for a hypothetical system, labeling exactly which security group rules connect each tier.
5. Write a Node.js script that measures and logs DNS resolution time separately from total request time for 5 different domains, and compare the results.

---

## 36. Mini Project

**Build:** A network-topology-aware deployment plan and diagnostic toolkit for the Module 2 horizontally-scaled blog platform.

**Requirements:**
- Produce a Mermaid deployment diagram showing public subnet (load balancer), private subnet (app servers), and data subnet (database), with explicit security group rules labeled on each boundary.
- Write a Node.js diagnostic script that measures DNS resolution time, TCP connect time, and total request time separately for a given URL, logging all three.
- Document, in `NETWORKING.md`, the specific security improvement this topology provides over a flat, single-subnet deployment (referencing Section 19's diagram).

**Success criteria:** Running your diagnostic script against your own deployed (or local, simulated) API produces three distinct timing measurements, and you can explain which of the three would be affected by each of: a DNS TTL change, a CDN, and a database index optimization.

---

## 37. Advanced Project

**Build:** Extend the Mini Project with a real, testable multi-tier network configuration.

1. Using a cloud provider's free tier (or a local tool like Docker networks/`docker-compose` to simulate subnets), actually construct a topology with an isolated "database" network that the "app" containers can reach, but that has no route to/from an "internet"-simulated network.
2. Attempt to connect to your simulated database from outside the "app" network and document that it fails — this is your practical proof of the Section 19 security guarantee, not just a diagram claim.
3. Introduce an artificial network latency (using a tool like `tc` on Linux, or a Docker network delay plugin) to simulate a distant user, and measure how this affects your application's real observed response time — connecting Section 11's math to an empirical result.
4. Write a short incident-response document: pretend the Section 34 Scenario 1 audit finding was real in your test environment, and walk through exactly how you'd verify, fix, and confirm remediation using the tools from this module.

**Success criteria:** You have empirical (not just diagrammed) evidence that your private subnet is unreachable from outside, and a measured before/after latency comparison demonstrating the real cost of network distance — setting up the motivating problem for Module 4 (HTTP, HTTPS, TCP & UDP), which explains exactly what happens *within* each of those network hops at the protocol level.

---

## 38. Summary

- Every "request" arrow in a system design diagram hides a real, multi-step process: DNS resolution, IP routing, port targeting, and firewall evaluation.
- **DNS** trades freshness for efficiency via caching and TTL — a real, tunable trade-off with operational consequences during migrations.
- **NAT solves address scarcity, not security** — private IPs still require proper firewall rules.
- **Network-layer isolation** (private subnets with no internet route) is a stronger security guarantee than firewall rules alone.
- **Bandwidth and latency are independent properties** — diagnosing which one is actually the bottleneck requires measurement, not assumption.
- Geographic distance causes real, physics-bound latency that no amount of compute scaling (Module 2) can fix — only proximity-based solutions (CDN, multi-region) address it.

---

## 39. Revision Notes

- IP = street address · Port = apartment number · DNS = phone book
- DNS resolution chain: browser cache → OS cache → recursive resolver → root → TLD → authoritative NS
- TTL trade-off: low = fast propagation, more query load; high = efficient caching, slow propagation
- NAT ≠ security; it's an address-sharing mechanism
- Private subnet with no internet gateway route > firewall rule alone, for sensitive components
- Bandwidth = volume/time; Latency = time for one trip — diagnose which is actually the bottleneck
- Distance-caused latency requires proximity solutions (CDN/multi-region), not compute scaling

---

## 40. One-Page Cheat Sheet

```
SYSTEM DESIGN — MODULE 3 CHEAT SHEET
─────────────────────────────────────
IP address   → street address (WHERE)
Port         → apartment number (WHICH SERVICE)
DNS          → phone book (NAME → ADDRESS)
Router       → forwards between networks
Switch       → forwards within a network
NAT          → shares one public IP among many private IPs (NOT security)
Firewall     → rule-based traffic filter (the actual security layer)

DNS RESOLUTION CHAIN
  Browser cache → OS cache → Recursive resolver → Root → TLD → Authoritative NS

BANDWIDTH vs LATENCY
  Bandwidth = how much data per second (truck size)
  Latency   = how long one trip takes (trip duration)
  More bandwidth ≠ lower latency for a single request!

SECURITY GOLDEN RULE
  Private subnet, no internet route > firewall rule alone
  Never expose DB/admin ports with 0.0.0.0/0 source rules
  NAT is NOT a substitute for firewall rules

MIGRATION CHECKLIST
  ☐ Lower DNS TTL in advance of planned IP/infra changes
  ☐ Restore TTL after migration confirmed stable
```

---

## Key Takeaways

- Networking fundamentals are the invisible substrate every later module (load balancing, CDN, API gateway, security) is built on — vague networking knowledge produces vague architecture reasoning downstream.
- The strongest security posture removes the network *path* to sensitive components entirely, rather than relying solely on rules that filter traffic on an existing path.
- Distance causes real latency that no compute-layer fix can solve — recognizing *which layer* a symptom belongs to is the actual skill being tested.

## 20 Practice Questions
*(See Section 33 — 6 Easy, 6 Medium, 5 Hard — plus 3 rapid-fire additions:)*
18. Why is `traceroute` a useful diagnostic tool, and what does a large latency jump between two specific hops suggest?
19. What's the practical difference between a public subnet and a private subnet in a cloud VPC?
20. Why should idempotency keys matter more in systems operating over unreliable public networks?

## 10 Scenario-Based Questions
*(See Section 34 in full.)*

## 5 Design Assignments
*(See Sections 36–37 — Mini Project and Advanced Project — plus:)*
1. Diagram the full network path (DNS through response) for a real website of your choice, using actual `dig` and `traceroute` output.
2. Write a one-page explanation of why NAT is not security, using an analogy of your own choosing.
3. Propose a DNS TTL strategy (with specific numbers) for a hypothetical service planning a major infrastructure migration in 2 weeks.

## Suggested Next Module

**→ Module 4: HTTP, HTTPS, TCP & UDP** — now that we understand how packets find their way to the correct machine and port, we go one level deeper into *what actually happens* during that connection: the TCP handshake, TLS encryption, and the evolution from HTTP/1.1 through HTTP/2, HTTP/3, WebSockets, and gRPC.
