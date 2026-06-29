# Day 11: RDS Overview — MySQL/PostgreSQL Managed Databases
### *(Why You Should (Almost) Never Run Your Own Database Server)*

> **Roadmap reference:** Week 2, Day 11 — "RDS Overview, MySQL/PostgreSQL"

---

## Why This Matters

You could absolutely install PostgreSQL directly on an EC2 instance — many beginners do exactly that as a first instinct. **RDS exists to talk you out of it.** Today is conceptual — understanding *why* RDS exists and what it actually manages for you — before you create a real instance tomorrow.

---

## 1. What is RDS?

**RDS (Relational Database Service) is AWS's managed database service — it runs, maintains, patches, backs up, and scales relational databases for you, so you don't have to administer the database server yourself.**

```
Self-Managed Database (DIY on EC2)     RDS (Managed)
───────────────────────────────────    ──────────────
You install PostgreSQL yourself     vs AWS provisions it for you
You patch OS + database software       AWS patches both automatically
You configure backups manually          Automated daily backups built-in
You handle failover yourself             Multi-AZ failover, automatic
You monitor disk space/performance       CloudWatch integration built-in
You scale by manually resizing            Scale storage/compute via console
```

### Supported Database Engines

```
RDS supports several engines, including:
  - PostgreSQL    ← your primary stack
  - MySQL
  - MariaDB
  - Oracle
  - SQL Server
  - Amazon Aurora (AWS's own high-performance MySQL/PostgreSQL-compatible engine)
```

> For your work (Node.js + PostgreSQL, as used in PulseBloom and QueueCare), you'd choose the **PostgreSQL** engine — RDS just hosts and manages a real, standard PostgreSQL instance; your application code, queries, and ORM (Prisma, etc.) work exactly the same as a self-hosted Postgres.

---

## 2. What RDS Actually Manages For You

```
┌────────────────────────────────────────────┐
│           WHAT RDS HANDLES                    │
├────────────────────────────────────────────┤
│  ✓ OS-level patching                          │
│  ✓ Database engine version patching            │
│  ✓ Automated daily backups                     │
│  ✓ Point-in-time recovery                       │
│  ✓ Multi-AZ failover (high availability)         │
│  ✓ Storage auto-scaling (optional)                │
│  ✓ Read replica creation                          │
│  ✓ Monitoring integration (CloudWatch)             │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│        WHAT'S STILL YOUR JOB                  │
├────────────────────────────────────────────┤
│  - Schema design                                │
│  - Query optimization                            │
│  - Application-level connection management        │
│  - IAM/Security Group access control                │
│  - Choosing the right instance size for your load   │
└────────────────────────────────────────────┘
```

> This is the Day 4 (Shared Responsibility Model) lesson, applied again here — AWS manages more of the "infrastructure" layer with a managed service like RDS than it would with a self-installed database on EC2.

---

## 3. Multi-AZ Deployments — High Availability for Your Database

Recall Day 2's Availability Zones lesson — RDS puts that concept to direct, practical use.

```
SINGLE-AZ (default, cheaper):
  Primary DB instance in ONE Availability Zone
        ↓
  AZ has an outage → database goes down → app goes down

MULTI-AZ (production-recommended):
  Primary DB in AZ-1a  ←──synchronous replication──→  Standby in AZ-1b
        ↓
  AZ-1a has an outage
        ↓
  RDS automatically fails over to the standby in AZ-1b
        ↓
  Your app reconnects (often within ~60-120 seconds) — no manual action needed
```

```
            Region: ap-south-1
   ┌─────────────────┐      ┌─────────────────┐
   │   AZ-1a            │      │   AZ-1b            │
   │  ┌─────────────┐ │      │  ┌─────────────┐ │
   │  │  PRIMARY DB    │ │◄────►│  │  STANDBY DB    │ │
   │  │  (active)       │ │ sync │  │  (passive,       │ │
   │  └─────────────┘ │ repl │  │   ready to take   │ │
   │                      │      │   over instantly)  │ │
   └─────────────────┘      └─────────────────┘
```

> **Free Tier note:** Single-AZ `db.t2.micro`/`db.t3.micro` is what's covered under the Free Tier (from Day 2's billing lesson). Multi-AZ roughly doubles cost, since you're paying for two running database instances — worth understanding for real production decisions, but for tomorrow's hands-on, you'll likely start Single-AZ to stay within Free Tier.

---

## 4. Read Replicas — Scaling Read-Heavy Workloads

A different problem than high availability — this is about **performance under read-heavy traffic.**

```
Without Read Replicas:
  ALL queries (reads AND writes) hit ONE database instance
        ↓
  Heavy read traffic (e.g., thousands of users browsing
  PulseBloom's community feed) competes with write traffic
  (e.g., users saving new mood entries) for the same resources

With Read Replicas:
  Writes  ──────────────► Primary DB
  Reads   ──────────────► Read Replica(s)  (one or more copies,
                                              asynchronously updated)
```

```
        Application
           │
    ┌──────┴──────┐
    │              │
  WRITES         READS
    │              │
    ▼              ▼
 PRIMARY DB    READ REPLICA
  (source        (copy, slightly
   of truth)      behind primary)
```

> **Important distinction from Multi-AZ:** A Multi-AZ standby is for *failover*, not normal traffic — it's not actively serving reads in standard Multi-AZ setups. A Read Replica is a *separate, actively queryable* copy, specifically meant to offload read traffic. Some setups use both together for serious production scale.

---

## 5. Automated Backups & Point-in-Time Recovery

```
RDS automatically:
  - Takes daily full backups during a configurable backup window
  - Continuously captures transaction logs
        ↓
  This combination allows "Point-in-Time Recovery" —
  restoring your database to ANY specific second within
  your retention window (commonly 7-35 days), not just to
  the last daily snapshot.
```

```
Example scenario:
  2:00 PM - A bad migration script accidentally deletes
            thousands of user records
  2:15 PM - You realize the mistake
        ↓
  You can restore the database to its exact state at 1:59 PM,
  just before the bad script ran — not just "yesterday's backup."
```

You'll set this up hands-on properly on **Day 14 (Backup and Snapshots)** — for now, just understand that RDS gives you this safety net automatically, unlike a self-managed database where you'd need to build this yourself.

---

## 6. Instance Classes — Sizing Your Database

Same underlying idea as EC2 Instance Types from Day 4, applied to database hardware:

```
db.t3.micro    → 2 vCPU (burstable), 1 GB RAM   → Free Tier, learning/dev
db.t3.medium   → 2 vCPU (burstable), 4 GB RAM   → small production apps
db.m5.large    → 2 vCPU (dedicated), 8 GB RAM   → real production traffic
```

> "Burstable" (the `t` family) instances are cost-efficient for workloads that aren't constantly under heavy load — fine for most early-stage apps like a first version of QueueCare or PulseBloom.

---

## 7. RDS Networking — Where It Lives, Security-Wise

```
RDS instances live INSIDE a VPC (you'll go deep on VPCs Day 15),
typically in PRIVATE subnets — meaning they should NOT be
directly reachable from the public internet at all.

        Internet
            │
            ▼
     EC2 Instance (public subnet)
       — your Express app —
            │
            │  only this can reach the DB,
            │  via Security Group rules
            ▼
     RDS Instance (PRIVATE subnet)
       — never exposed publicly —
```

> This is the same Security Group logic from Day 5, applied to a database: the RDS instance's Security Group should only allow inbound traffic on port 5432 (PostgreSQL's default port) **from your application's Security Group** — never from `0.0.0.0/0`. A database open to the entire internet is an even more severe version of the open-SSH mistake from Day 4/5.

---

## 8. Connecting This to Your Own Experience

This maps directly onto real decisions you've already had to make while deploying PulseBloom and QueueCare:

```
Your past debugging around RDS/SSL configuration and Prisma
adapter patterns connects directly to today's concepts:

  SSL configuration issues  → RDS enforces encrypted connections
                                by default; your app's connection
                                string/ORM config must match that

  Prisma v7 adapter patterns → how your ORM establishes and pools
                                 connections to a managed RDS
                                 PostgreSQL instance specifically

  ECS health checks failing   → often traced back to the ECS task's
   on DB connectivity            Security Group not being allowed
                                  through to RDS's Security Group
```

Today's conceptual grounding is exactly what explains *why* those past real-world issues happened the way they did.

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is RDS, and why use it instead of installing a database on EC2 yourself?**
> RDS is AWS's managed relational database service — it handles patching, backups, failover, and monitoring automatically. Self-managing a database on EC2 means you're responsible for all of that manually, which is significant operational overhead, especially for production systems.

**Q2: What's the difference between Multi-AZ and a Read Replica?**
> Multi-AZ provides high availability via a synchronously-replicated standby that automatically takes over during a failure — it's for failover, not normal traffic. A Read Replica is an actively queryable copy used to offload read traffic and improve performance, not primarily for failover.

**Q3: What is Point-in-Time Recovery?**
> The ability to restore an RDS database to any specific moment within your backup retention window, using continuous transaction logs combined with daily backups — not just restoring to the last full snapshot.

**Q4: Should an RDS instance ever be placed in a public subnet, directly reachable from the internet?**
> No — best practice is placing RDS in a private subnet, only reachable from your application's Security Group (e.g., your EC2/ECS instances), never directly from the public internet.

**Q5: What database engines does RDS support?**
> PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, and Amazon Aurora (AWS's own high-performance, MySQL/PostgreSQL-compatible engine).

---

## 10. Hands-On Assignment for Today (Exploration Only — No Creation Yet)

1. Go to **AWS Console → RDS** (don't create anything yet — that's tomorrow).
2. Click **"Create database"** to open the configuration screen, but **don't submit it** — just explore the options:
   - Engine options (note PostgreSQL)
   - Templates (Free Tier option)
   - Instance class options
   - Multi-AZ toggle
3. Note where the **VPC/subnet group** and **Security Group** settings appear in this flow — you'll configure them properly tomorrow.
4. Write a short note in your own words covering:
   - Why RDS is generally preferred over self-hosting a database on EC2
   - The difference between Multi-AZ and Read Replicas
   - Why an RDS instance should sit in a private subnet, not a public one

---

## 11. Day 11 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"RDS is AWS's managed database service — it handles patching, automated backups, and failover so I don't have to. Multi-AZ gives me high availability through a standby in a different Availability Zone, while Read Replicas scale read-heavy traffic with actively queryable copies. My database should always live in a private subnet, only reachable from my application's Security Group — never directly exposed to the internet."*

---

**Next up — Day 12: Create a PostgreSQL RDS Instance (Hands-On)**
