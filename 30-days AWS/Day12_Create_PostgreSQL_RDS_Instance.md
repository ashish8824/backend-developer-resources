# Day 12: Create a PostgreSQL RDS Instance
### *(From Concept to a Real Running Database in AWS)*

> **Roadmap reference:** Week 2, Day 12 — "Create PostgreSQL RDS Instance"

---

## Why This Matters

Yesterday was the "why" — today is the "how." You'll actually provision a managed PostgreSQL database in AWS, configure its networking and security correctly, and connect to it from your terminal. This is the exact workflow you'd follow when setting up a database for a real backend project like QueueCare or PulseBloom from scratch.

---

## 1. The Plan for Today

```
Step 1: Create a Security Group for RDS
        (controls who can talk to the database)

Step 2: Create the RDS PostgreSQL Instance
        (the actual database server)

Step 3: Connect to it from your EC2 instance
        (verifying everything works end-to-end)

Step 4: Run basic SQL commands
        (confirming it's a real, working PostgreSQL database)
```

---

## 2. Step 1: Create a Dedicated Security Group for RDS

Before creating the database, set up its firewall first — doing it in this order helps you understand what you're configuring and why.

```
AWS Console → EC2 → Security Groups → "Create security group"

  Name: rds-postgres-sg
  Description: Security group for PostgreSQL RDS instance
  VPC: default VPC (for now)

  Inbound Rules:
    Type:        PostgreSQL
    Port:        5432
    Source:      Custom → [select your EC2 instance's Security Group]
                 (NOT 0.0.0.0/0 — your database should only accept
                  connections from your application, never from the
                  whole internet)

  Outbound Rules: leave as default (all traffic allowed)

  → Create security group
```

```
Why source = your EC2's Security Group, not a specific IP?
  IP-based restriction:
    If your EC2 instance gets stopped and restarted, its private
    IP might change — your rule would need updating every time.

  Security Group-based restriction (what we're doing):
    AWS resolves this dynamically — "allow any instance that has
    this Security Group attached" — survives IP changes cleanly.
    This is best practice for app-to-database rules.
```

---

## 3. Step 2: Create the RDS PostgreSQL Instance

```
AWS Console → RDS → "Create database"

  Choose a database creation method:
    → Standard create

  Engine options:
    → PostgreSQL
    → Version: latest LTS available (e.g., PostgreSQL 15.x or 16.x)

  Templates:
    → Free tier
      (this automatically sets Single-AZ and db.t3.micro,
       which is exactly what we want for learning without cost)

  Settings:
    DB instance identifier: my-first-rds-db
    Master username:         postgres
    Master password:         [choose something strong, save it now]
    Confirm password:        [same]

  Instance configuration:
    → db.t3.micro (pre-selected by Free Tier template)

  Storage:
    → Allocated storage: 20 GB (Free Tier default)
    → Storage autoscaling: disable for now (prevents surprise cost growth)

  Connectivity:
    → VPC: default
    → Subnet group: default
    → Public access: NO
        ← this is important — your database should NOT be
           publicly accessible from the internet
    → VPC security groups: choose "existing"
        → Remove the default one, add: rds-postgres-sg
          (the one you just created in Step 1)

  Additional configuration:
    → Initial database name: appdb
        (if you leave this blank, RDS creates no default database —
         setting it now is convenient)
    → Backup retention period: 7 days (keep this — it's your safety net)
    → Leave everything else as default

  → Create database
```

```
⏳ Creation takes approximately 3-5 minutes.
   The status will show "Creating" then "Available".
   Don't refresh aggressively — just wait.
```

---

## 4. Finding Your RDS Endpoint

Once the status shows **"Available"**, this is the most important piece of information to find:

```
RDS → Databases → click your instance → "Connectivity & security" tab

  Endpoint:  my-first-rds-db.xxxxxxxxxx.ap-south-1.rds.amazonaws.com
  Port:      5432
```

**The endpoint is the "address" of your database** — this is what you'll put in your application's connection string instead of `localhost:5432`.

```
Local PostgreSQL connection string:
  postgresql://postgres:password@localhost:5432/appdb

RDS connection string:
  postgresql://postgres:password@my-first-rds-db.xxxxxxxxxx.ap-south-1.rds.amazonaws.com:5432/appdb
                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                  this hostname is the only thing that changes
```

---

## 5. Step 3: Connect to RDS from Your EC2 Instance

Your RDS instance is in a **private subnet** with no public access — so you can't connect directly from your laptop. You connect through your EC2 instance, which is in a public subnet and has the allowed Security Group.

```
Your Laptop  →  SSH  →  EC2 Instance  →  TCP 5432  →  RDS Instance
(can't reach RDS       (public subnet,    (private subnet,
 directly)              has allowed SG)    only allows EC2's SG)
```

### Install the PostgreSQL Client on EC2

```bash
# SSH into your EC2 instance first
ssh -i my-first-server-key.pem ubuntu@<your-elastic-ip>

# Install just the PostgreSQL client (not a full server)
sudo apt update
sudo apt install -y postgresql-client

# Verify
psql --version
```

### Connect to Your RDS Instance

```bash
psql \
  --host=my-first-rds-db.xxxxxxxxxx.ap-south-1.rds.amazonaws.com \
  --port=5432 \
  --username=postgres \
  --dbname=appdb
```

```
Password prompt appears → enter your master password
        ↓
appdb=#                 ← you're now inside PostgreSQL on RDS
```

---

## 6. Step 4: Run Basic SQL Commands to Confirm It Works

```sql
-- Check which database you're in
SELECT current_database();

-- Check PostgreSQL version
SELECT version();

-- Create a test table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Insert a test row
INSERT INTO users (name, email)
VALUES ('Ashish', 'ashish@example.com');

-- Query it back
SELECT * FROM users;

-- Expected output:
--  id |  name   |        email         |         created_at
-- ----+---------+----------------------+----------------------------
--   1 | Ashish  | ashish@example.com   | 2026-06-22 10:00:00.000

-- Clean up
DROP TABLE users;

-- Exit
\q
```

If you see that `SELECT` result, **your RDS instance is fully working** — a real managed PostgreSQL database running in AWS, connected from your application tier.

---

## 7. What You Just Built — Architecture Check

```
                    Internet
                        │
                        │  (SSH — your laptop only)
                        ▼
          ┌─────────────────────────────┐
          │  EC2 Instance (public subnet) │
          │  — your Express app tier —    │
          │  Security Group: ec2-sg        │
          └─────────────────────────────┘
                        │
                        │  Port 5432 (PostgreSQL)
                        │  EC2's sg → allowed in rds-postgres-sg
                        ▼
          ┌─────────────────────────────┐
          │  RDS Instance (private subnet)│
          │  — PostgreSQL database —       │
          │  Security Group: rds-postgres-sg│
          │  Public Access: NO              │
          └─────────────────────────────┘
```

This two-tier architecture (public app tier + private data tier) is the standard pattern for every serious backend deployment — not just an exam concept but genuinely how production systems are laid out.

---

## 8. SSL Connection (Important for Production)

RDS enforces SSL connections by default, which is exactly what you want for data-in-transit encryption — but your application must be configured to handle it. For now, just be aware:

```bash
# If you get an SSL warning connecting from psql, you can verify
# the SSL mode in use like this:
psql "host=<endpoint> port=5432 dbname=appdb user=postgres \
  sslmode=require"
```

```
When connecting from your Node.js app (Tomorrow, Day 13):
  Prisma / pg connection string with SSL:
    postgresql://postgres:password@<endpoint>:5432/appdb?sslmode=require

  If your local pg client needs SSL disabled for local dev
  but RDS requires it for production, you'd control this
  via environment variables — not hardcoded in code.
```

---

## 9. Cost Awareness — Free Tier + Stopping Your Instance

```
Free Tier gives you: 750 hours/month of db.t3.micro (Single-AZ)
  → enough to run ONE instance continuously for an entire month

BUT:
  RDS does NOT have a simple "Stop temporarily" button like EC2
  → You can stop it for up to 7 days (then it auto-restarts)
  → After 7 days, AWS restarts it automatically to apply patches

Best practice for learning:
  Use it actively for the week (Days 12-14), then STOP it
  (RDS → Actions → Stop temporarily) — you won't be charged
  for the instance compute hours while it's stopped,
  though you still pay for the storage (20GB ≈ very small).
```

---

## 10. Common Errors and Fixes

```
Error: "Connection timed out" when connecting from EC2
  Fix: Check that rds-postgres-sg inbound rule actually references
       your EC2 instance's Security Group (not its IP, not 0.0.0.0/0)

Error: "Password authentication failed for user postgres"
  Fix: Double-check the master password — it's case-sensitive.
       If forgotten, you CAN reset it via RDS → Modify Instance.

Error: "SSL connection required"
  Fix: Add sslmode=require to your connection string

Error: "database appdb does not exist"
  Fix: You left "Initial database name" blank during creation.
       Connect without specifying a dbname, then CREATE DATABASE appdb;

Error: psql command not found
  Fix: sudo apt install -y postgresql-client on EC2
```

---

## 11. Interview Questions (Practice Explaining These Out Loud)

**Q1: Walk me through how you'd create a production-ready RDS instance.**
> Choose PostgreSQL engine, Free Tier or appropriate instance class, disable public access, place in a private subnet, attach a Security Group that only allows inbound 5432 from the application tier's Security Group, enable automated backups with a 7+ day retention, and note the endpoint for use in the application's connection string.

**Q2: Why should Public Access be set to "No" for an RDS instance?**
> A database with public access enabled is directly reachable from the internet on port 5432, making it vulnerable to brute-force and exploitation attempts. Databases should only be reachable from the application layer, within the same VPC.

**Q3: How is an RDS endpoint different from `localhost` in a connection string?**
> `localhost` refers to the same machine running the app. An RDS endpoint is a DNS hostname pointing to the managed database instance, which may be on entirely different infrastructure — but from the application's perspective, only the hostname in the connection string changes.

**Q4: Why use a Security Group reference (rather than an IP address) as the inbound source for RDS?**
> Instance IPs can change on restart. Referencing another Security Group means "allow any resource that has this Security Group attached," which dynamically resolves correctly regardless of IP changes.

**Q5: What's the Free Tier limit for RDS?**
> 750 hours per month of db.t2.micro/db.t3.micro (Single-AZ) for the first 12 months — enough to run one Free Tier instance continuously for a full month.

---

## 12. Hands-On Assignment for Today

1. Create the **rds-postgres-sg** Security Group (inbound 5432 from your EC2's SG only).
2. Create a **PostgreSQL RDS instance** using Free Tier settings, Public Access: No, Security Group: rds-postgres-sg.
3. Wait for status → **"Available"**, then copy the endpoint.
4. SSH into your EC2, install `postgresql-client`, and connect using `psql`.
5. Run the SQL block from Section 6 — `CREATE TABLE`, `INSERT`, `SELECT`, confirm results.
6. **Stop the RDS instance** when done (Actions → Stop temporarily) to preserve Free Tier hours.
7. Write a short note in your own words covering:
   - Why you used a Security Group reference instead of an IP for inbound rules
   - The difference between an RDS endpoint and localhost
   - What "Public Access: No" actually protects against

---

## 13. Day 12 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"I created a PostgreSQL RDS instance in a private subnet with no public internet access, protected by a Security Group that only allows inbound traffic from my EC2 instance's Security Group on port 5432. I connected through EC2 using psql, confirmed the database works, and know that my application just needs to replace 'localhost' with the RDS endpoint in its connection string — everything else stays the same."*

---

**Next up — Day 13: Connect Node.js App to RDS**
