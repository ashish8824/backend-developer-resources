# Day 16: Internet Gateway & Route Tables
### *(The Two Things That Actually Connect Your VPC to the Internet)*

> **Roadmap reference:** Week 3, Day 16 — "Internet Gateway, Route Tables"

---

## Why This Matters

Yesterday you built the skeleton of a custom VPC — subnets created, but nothing works yet. If you launched an EC2 instance into one of those public subnets right now, it would have no internet access at all. No SSH, no browser traffic, nothing. That's because **creating a subnet doesn't automatically connect it to the internet** — that requires two more pieces working together: an Internet Gateway and a Route Table.

Today you add those two pieces and your custom VPC becomes fully functional.

---

## 1. The Problem — Why Your VPC Can't Reach the Internet Yet

```
Current state of your custom VPC from Day 15:

  VPC: 10.0.0.0/16
    ├── public-subnet-1a:  10.0.1.0/24
    ├── public-subnet-1b:  10.0.2.0/24
    ├── private-subnet-1a: 10.0.3.0/24
    └── private-subnet-1b: 10.0.4.0/24

  Problem: None of these subnets know HOW to route traffic
           to the internet yet. They're isolated islands.

  Missing pieces:
    1. Internet Gateway  → the physical "door" to the internet
    2. Route Table       → the "map" telling traffic which door to use
```

---

## 2. Internet Gateway (IGW) — The Door to the Internet

**An Internet Gateway is a horizontally scaled, redundant AWS-managed component that allows communication between your VPC and the internet.**

```
Internet Gateway key facts:
  → Attached to a VPC (one IGW per VPC)
  → Managed entirely by AWS — no uptime, scaling, or bandwidth
    limits on your end
  → Free to create and attach (you only pay for data transfer)
  → Does TWO jobs:
      1. Routes outbound traffic FROM your VPC TO the internet
      2. Allows inbound traffic FROM the internet TO reach
         resources in your public subnets (subject to Security
         Group rules — Day 5)
```

```
Without IGW:                  With IGW attached:
  VPC  ─── ✗ ──► Internet       VPC  ──► IGW ──► Internet
  (blocked, no route)            (open, traffic can flow both ways)
```

> **Analogy:** If your VPC is a gated residential community, the Internet Gateway is the main entrance/exit gate. Without it, nobody gets in or out, no matter how many internal roads (subnets) you've built inside.

### Creating and Attaching an Internet Gateway

```
Step 1: Create the IGW
  VPC → Internet Gateways → Create internet gateway
    Name: my-production-igw
    → Create

Step 2: Attach it to your VPC
  Select the new IGW → Actions → Attach to VPC
    → Select: my-production-vpc
    → Attach
```

```
Status changes from "Detached" → "Attached"
The IGW now exists as the gateway for your VPC,
but traffic still won't flow until Route Tables
tell resources HOW to use it.
```

---

## 3. Route Tables — The Traffic Map

**A Route Table is a set of rules (routes) that tells network traffic where to go, based on the destination IP address.**

```
Every subnet in your VPC must be associated with a Route Table.
The Route Table is what the subnet uses to decide:
  "Where do I send this packet?"
```

### Anatomy of a Route Table

```
Route Table: public-route-table
┌─────────────────┬────────────────────┬──────────┐
│ Destination      │ Target              │ Status   │
├─────────────────┼────────────────────┼──────────┤
│ 10.0.0.0/16      │ local               │ Active   │
│ 0.0.0.0/0        │ igw-xxxxxxxxxx      │ Active   │
└─────────────────┴────────────────────┴──────────┘
```

```
Breaking down those two routes:

Row 1: 10.0.0.0/16  →  local
  "Any traffic destined for an IP inside my VPC
   (10.0.0.0 to 10.0.255.255) stays local —
   route it internally within the VPC."
  → This route exists in EVERY route table automatically.
    It's how your EC2 can talk to your RDS without
    leaving the VPC.

Row 2: 0.0.0.0/0  →  igw-xxxxxxxxxx
  "ANY traffic not covered by a more specific route
   (i.e., anything destined for the public internet)
   should be sent through the Internet Gateway."
  → This is THE route that makes a subnet "public."
  → Private subnets do NOT have this route.
```

### The "Most Specific Route Wins" Rule

Route tables use **longest prefix matching** — the most specific matching route always wins:

```
Destination: 10.0.2.45 (inside your VPC)
  → Matches 10.0.0.0/16 (local) → stays inside VPC ✓

Destination: 13.234.50.10 (public internet)
  → Doesn't match 10.0.0.0/16
  → Matches 0.0.0.0/0 (catch-all) → goes through IGW ✓
```

---

## 4. Wiring It Together — Step by Step

### Create a Public Route Table

```
VPC → Route Tables → Create route table
  Name: public-route-table
  VPC: my-production-vpc
  → Create
```

### Add the Internet Gateway Route

```
Select public-route-table → Routes tab → Edit routes → Add route
  Destination: 0.0.0.0/0
  Target: Internet Gateway → select my-production-igw
  → Save changes
```

### Associate the Public Subnets

```
Select public-route-table → Subnet associations tab
  → Edit subnet associations
  → Check: public-subnet-1a
  → Check: public-subnet-1b
  → Save
```

```
Result: Any EC2 instance launched in public-subnet-1a or
        public-subnet-1b now has a path to the internet
        through the IGW — these subnets are now TRULY public.
```

### Create a Private Route Table (For Completeness)

```
VPC → Route Tables → Create route table
  Name: private-route-table
  VPC: my-production-vpc
  → Create

Associate it with:
  private-subnet-1a
  private-subnet-1b

Routes (only the auto-created local route):
┌─────────────────┬────────────────────┬──────────┐
│ Destination      │ Target              │ Status   │
├─────────────────┼────────────────────┼──────────┤
│ 10.0.0.0/16      │ local               │ Active   │
└─────────────────┴────────────────────┴──────────┘

No 0.0.0.0/0 → IGW route here.
Resources in private subnets have NO path to the internet.
```

> Note: Private subnets can still reach the internet for outbound requests (e.g., to download OS patches) via a **NAT Gateway** — you'll add that on Day 18. For now, private subnets are truly isolated.

---

## 5. The Full Custom VPC — What You've Built So Far

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  Internet Gateway (my-production-igw)                  │
└──────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  VPC: my-production-vpc (10.0.0.0/16)                  │
│                                                          │
│  Public Route Table                                      │
│    10.0.0.0/16 → local                                  │
│    0.0.0.0/0   → IGW  ←── this makes it PUBLIC         │
│         │                                                │
│  ┌──────┴──────────────────┐                           │
│  │  public-subnet-1a         │  public-subnet-1b        │
│  │  10.0.1.0/24 (AZ: 1a)    │  10.0.2.0/24 (AZ: 1b)  │
│  │  [EC2 goes here]          │  [Load Balancer here]    │
│  └───────────────────────────┘                          │
│                                                          │
│  Private Route Table                                     │
│    10.0.0.0/16 → local only ←── no IGW = PRIVATE       │
│         │                                                │
│  ┌──────┴──────────────────┐                           │
│  │  private-subnet-1a        │  private-subnet-1b       │
│  │  10.0.3.0/24 (AZ: 1a)    │  10.0.4.0/24 (AZ: 1b)  │
│  │  [RDS goes here]          │  [RDS standby here]      │
│  └───────────────────────────┘                          │
└──────────────────────────────────────────────────────┘
```

---

## 6. Connecting This to Everything So Far

```
Day 5 (Security Groups):
  "Security Groups control which ports/IPs can reach your instance."
        ↓
  But Security Groups only apply AFTER traffic has already been
  routed to your subnet via the Route Table. Route Tables decide
  IF traffic can reach the subnet at all. Security Groups decide
  IF traffic can reach the specific instance.

Day 12 (RDS in private subnet):
  "Set Public Access: NO on RDS."
        ↓
  Now you understand WHY this works — the private subnet's
  Route Table has no IGW route, so there's literally no path
  for internet traffic to reach your database, regardless of
  Security Group settings.

Day 15 (subnets):
  "What makes a subnet public or private is its Route Table."
        ↓
  Today you proved this by actually adding the IGW route to
  the public Route Table and watching it take effect.
```

---

## 7. Common Mistakes and How to Spot Them

```
Mistake 1: Created the IGW but forgot to ATTACH it to the VPC
  Symptom: Route table has 0.0.0.0/0 → IGW, but IGW status
            still shows "detached"
  Fix: IGW → Actions → Attach to VPC

Mistake 2: Added the IGW route to the Route Table but forgot
           to ASSOCIATE the subnet with that Route Table
  Symptom: EC2 in your subnet still can't reach internet
  Fix: Route Table → Subnet associations → add the subnet

Mistake 3: Associated the public subnet with the PRIVATE
           Route Table by mistake (or vice versa)
  Symptom: EC2 in "public" subnet has no internet, RDS in
            "private" subnet accidentally has internet access
  Fix: Check every subnet's Route Table association carefully

Mistake 4: No 0.0.0.0/0 route in the public Route Table
  Symptom: Even with IGW attached and associated, instance
            can't reach internet
  Fix: Route Table → Routes → Edit → Add 0.0.0.0/0 → IGW
```

---

## 8. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is an Internet Gateway and what does it do?**
> An AWS-managed component attached to a VPC that provides a path for traffic to flow between the VPC and the public internet — both inbound and outbound, subject to Security Group rules.

**Q2: What is a Route Table?**
> A set of routing rules that tells network traffic where to go based on the destination IP. Every subnet must be associated with a Route Table; the Route Table determines whether the subnet is effectively public or private.

**Q3: What is the `0.0.0.0/0` route, and what does pointing it to an IGW do?**
> `0.0.0.0/0` is the catch-all route matching any IP address not covered by more specific routes — effectively "everything else." Pointing it to an Internet Gateway means all outbound internet traffic is routed through that gateway, making the subnet public.

**Q4: Can a VPC have more than one Internet Gateway?**
> No — one IGW per VPC. However, one IGW handles traffic for all public subnets inside that VPC simultaneously. AWS manages the scaling and redundancy of the IGW itself.

**Q5: What's the difference between how a Route Table controls access vs how a Security Group controls access?**
> A Route Table controls whether a network PATH exists to reach the subnet at all — it's network-level routing. A Security Group controls whether specific traffic is allowed to reach a specific instance — it's instance-level firewall. Both must allow traffic for it to get through; if either blocks it, the traffic is stopped.

**Q6: If a private subnet has no 0.0.0.0/0 route, can resources in it initiate outbound internet connections?**
> Not by default. To allow private subnet resources to make outbound calls (e.g., downloading patches, calling external APIs) without being reachable FROM the internet, you add a NAT Gateway — which you'll cover on Day 18.

---

## 9. Hands-On Assignment for Today

1. **Create an Internet Gateway** (`my-production-igw`) and attach it to `my-production-vpc`.

2. **Create a public Route Table** (`public-route-table`):
   - Add route: `0.0.0.0/0 → my-production-igw`
   - Associate with: `public-subnet-1a` and `public-subnet-1b`

3. **Create a private Route Table** (`private-route-table`):
   - Leave only the auto-created local route
   - Associate with: `private-subnet-1a` and `private-subnet-1b`

4. **Launch a test EC2 instance** in `public-subnet-1a` of your custom VPC:
   - Enable auto-assign public IP
   - Use a Security Group allowing SSH from your IP
   - SSH into it and run `curl https://api.ipify.org` — confirm it returns a public IP (proves internet access works)

5. **Terminate the test instance** when done.

6. Write a short note covering:
   - What a Route Table route of `0.0.0.0/0 → IGW` actually means
   - The order of evaluation: Route Table first, then Security Group
   - Why private subnets don't have the IGW route

---

## 10. Day 16 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"An Internet Gateway is the door between my VPC and the internet — but a door alone doesn't route traffic. A Route Table is the map that tells traffic which door to use. By adding a `0.0.0.0/0 → IGW` route to a Route Table and associating it with my public subnets, those subnets can now send and receive internet traffic. Private subnets have no such route — there's literally no path for internet traffic to reach resources in them, which is why my RDS is safe there. Security Groups add a second layer of control after the route table already decided traffic can reach the subnet at all."*

---

**Next up — Day 17: Public vs Private Subnets (Hands-On Deep Dive)**
