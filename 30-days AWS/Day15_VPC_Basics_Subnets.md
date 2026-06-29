# Day 15: VPC Basics & Subnets
### *(Your Own Private Network Inside AWS — Understanding the Foundation of Everything)*

> **Roadmap reference:** Week 3, Day 15 — "VPC Basics, Subnets"

---

## Why This Matters

Every single resource you've created so far — EC2, RDS, even S3 connections — has been living inside a **VPC**, whether you realized it or not. You've been using the "default VPC" that AWS creates automatically for every new account, without thinking about it.

Week 3 is about **Networking** — and VPC is the foundation of all of it. Understanding VPC properly explains *why* your RDS lives in a private subnet, *why* your EC2 can reach the internet but RDS cannot, and *how* you'd design a secure, production-grade network from scratch. Every senior backend/cloud developer interview will touch this.

---

## 1. What is a VPC?

**A VPC (Virtual Private Cloud) is your own logically isolated, private network inside AWS — where you control the IP address ranges, subnets, routing, and network access for all your resources.**

```
Without VPC (impossible in real AWS — but imagine it):
  All AWS customers' resources exist in one giant shared network
  → any EC2 instance could potentially reach any other customer's
    EC2 instance
  → complete security nightmare

With VPC:
  Your resources live in YOUR private network
  → completely isolated from other AWS customers' networks
  → you define who can talk to whom, using your own rules
```

> **Analogy:** Think of AWS as a massive office building with thousands of companies inside. A VPC is your company's **private floor** — you control the layout, the internal rooms, the locks on each door, and who's allowed in from the outside. Other companies on other floors can't see or reach your network by default.

---

## 2. CIDR Blocks — Defining Your Network's IP Range

Before subnets make sense, you need to understand **CIDR (Classless Inter-Domain Routing)** — the notation used to define IP address ranges.

```
Example CIDR block: 10.0.0.0/16

Breaking it down:
  10.0.0.0    → the starting IP address of your network
  /16          → the first 16 bits are fixed (the "network" part)
                 the remaining 16 bits are flexible (the "host" part)

How many IPs does /16 give you?
  32 total bits - 16 fixed = 16 flexible bits
  2^16 = 65,536 IP addresses (10.0.0.0 → 10.0.255.255)
```

```
Common CIDR sizes and what they mean:

  /16 → 65,536 IPs  (large VPC — typical for a whole company)
  /24 → 256 IPs     (typical subnet size)
  /28 → 16 IPs      (small subnet, e.g., for a NAT gateway)

Quick memory trick:
  The bigger the number after /, the SMALLER the network
  /16 = huge     /28 = tiny
```

> You don't need to be a CIDR math expert right now — just understand that when AWS asks for a CIDR block, you're defining the pool of IP addresses available in that network or subnet.

---

## 3. VPC Structure — The Full Picture

```
AWS Account
    │
    └── VPC: 10.0.0.0/16  (your private network, 65,536 IPs)
              │
              ├── PUBLIC SUBNET: 10.0.1.0/24   (256 IPs)
              │     → Resources here CAN reach the internet
              │     → Internet CAN reach resources here
              │     → Typical occupants: EC2 instances,
              │                          Load Balancers,
              │                          NAT Gateways
              │
              └── PRIVATE SUBNET: 10.0.2.0/24  (256 IPs)
                    → Resources here CANNOT be reached from internet
                    → Resources here CAN reach internet (via NAT)
                    → Typical occupants: RDS databases,
                                         ElastiCache,
                                         Internal microservices
```

---

## 4. Subnets — Dividing Your VPC into Segments

**A Subnet is a subdivision of your VPC's IP range — a smaller network range assigned to a specific Availability Zone, with its own routing rules.**

### The Critical Rule: One Subnet = One AZ

```
VPC spans ALL Availability Zones in a Region
        │
        ├── Subnet A → pinned to AZ-1a  (can only contain resources in AZ-1a)
        ├── Subnet B → pinned to AZ-1b
        └── Subnet C → pinned to AZ-1c
```

> This is why creating resources in "multiple AZs" (for high availability, from Day 2) means creating resources in multiple *subnets* — one per AZ. The subnets are the physical AZ anchors.

### Public vs Private Subnet — What Actually Makes the Difference?

This is the single most misunderstood point about subnets — beginners often think "public" or "private" is a setting you flip on the subnet. It's not. **What makes a subnet public or private is whether its Route Table routes traffic through an Internet Gateway.**

```
Public Subnet:
  Route Table entry:
    0.0.0.0/0  →  Internet Gateway (igw-xxxxxxxx)
    ↑
  "Send all outbound traffic to the internet"
  Resources here can SEND and RECEIVE internet traffic.

Private Subnet:
  Route Table entry:
    0.0.0.0/0  →  NAT Gateway (or nothing)
    ↑
  No Internet Gateway in the route
  Resources here CANNOT be directly reached from the internet.
  They CAN initiate outbound internet traffic via NAT (Day 18).
```

You'll go deeper into Internet Gateway and Route Tables on **Day 16** — for now, just absorb this key point: **the route table is what defines public vs private**, not any subnet setting itself.

---

## 5. Default VPC vs Custom VPC

Every AWS account comes with a **Default VPC** pre-created in every region:

```
Default VPC (what you've been using):
  CIDR: 172.31.0.0/16
  Has pre-created public subnets in every AZ
  Everything is public by default
  → Fine for learning and quick testing
  → NOT acceptable for production (everything public = security risk)

Custom VPC (what real apps use):
  You define the CIDR block
  You design the subnet layout
  You control exactly what's public vs private
  → Required for any production system
```

> Everything you've built in Weeks 1–2 used the default VPC — that's why it was quick to get running. Week 3 teaches you how to design a proper one.

---

## 6. Designing a Real-World VPC Layout

Here's the subnet design used by most serious production Node.js backends on AWS — including setups similar to PulseBloom/QueueCare:

```
VPC: 10.0.0.0/16
│
├── PUBLIC SUBNETS (for internet-facing resources)
│     ├── public-subnet-1a: 10.0.1.0/24  (AZ: ap-south-1a)
│     └── public-subnet-1b: 10.0.2.0/24  (AZ: ap-south-1b)
│                 ↑
│     EC2 instances, Load Balancers, NAT Gateways live here
│
└── PRIVATE SUBNETS (for backend/data resources)
      ├── private-subnet-1a: 10.0.3.0/24  (AZ: ap-south-1a)
      └── private-subnet-1b: 10.0.4.0/24  (AZ: ap-south-1b)
                  ↑
      RDS databases, ElastiCache, internal services live here
```

```
Two AZs for everything → High availability (Day 2's lesson, now made concrete)
Public + Private split → Security (databases never directly internet-reachable)
```

---

## 7. How Everything You've Built Maps to This

```
Weeks 1–2 resources, now mapped to VPC concepts:

  EC2 Instance (Day 4)
    → Lives in a PUBLIC subnet
    → Has a public IP + Elastic IP
    → Reachable from internet (your Nginx on port 80)

  RDS Instance (Day 12)
    → Lives in a PRIVATE subnet
    → No public IP (Public Access: NO)
    → NOT reachable from internet
    → Only reachable from EC2's Security Group

  S3 (Day 8-10)
    → S3 is a global service, NOT inside your VPC
    → Accessed via AWS's backbone or a VPC Endpoint
      (advanced concept — for now, your EC2 reaches
       it over the internet via the SDK)
```

> This is why setting `Public Access: NO` on RDS on Day 12 was the right call — your RDS is in a private subnet by design, and that setting enforces it even if someone accidentally misconfigures a route table later.

---

## 8. Key VPC Components — Preview Map

You'll learn each of these in detail over the next few days — here's how they connect so you have the full picture before diving in:

```
┌─────────────────────────────────────────────────────────┐
│  VPC (10.0.0.0/16)                                       │
│                                                           │
│  ┌──────────────────────┐  ┌─────────────────────────┐  │
│  │   PUBLIC SUBNET        │  │    PRIVATE SUBNET         │  │
│  │   10.0.1.0/24          │  │    10.0.3.0/24            │  │
│  │                          │  │                           │  │
│  │  [EC2]  [Load Balancer] │  │   [RDS]  [ElastiCache]   │  │
│  │  [NAT Gateway]          │  │                           │  │
│  └──────────────────────┘  └─────────────────────────┘  │
│           │                              │                 │
│    Route Table                     Route Table             │
│    0.0.0.0/0 → IGW            0.0.0.0/0 → NAT GW         │
│           │                                                 │
│    ┌──────▼──────┐                                        │
│    │Internet Gateway│  ← Day 16                           │
│    └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
                    │
                Internet
```

```
Components you'll master this week:
  Day 15 (today):  VPC, Subnets
  Day 16:          Internet Gateway, Route Tables
  Day 17:          Public vs Private Subnets (hands-on)
  Day 18:          Load Balancer
  Day 19:          Auto Scaling
  Day 20-21:       CloudWatch Monitoring
```

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is a VPC?**
> A logically isolated private network within AWS where you control IP address ranges, subnets, routing, and network access for all your resources — completely isolated from other AWS customers' networks.

**Q2: What is a subnet, and what's the relationship between a subnet and an Availability Zone?**
> A subnet is a subdivision of a VPC's IP range, assigned to a specific Availability Zone. One subnet can only exist in one AZ. To achieve multi-AZ high availability, you create subnets in multiple AZs.

**Q3: What actually makes a subnet "public" vs "private"?**
> Whether its Route Table routes outbound traffic through an Internet Gateway. If it does, instances in that subnet can send and receive internet traffic — making it "public." Without an Internet Gateway in the route, the subnet is effectively private.

**Q4: What's the difference between the Default VPC and a custom VPC?**
> The Default VPC is pre-created by AWS with all-public subnets — convenient for testing but not suitable for production since there's no private isolation. A custom VPC lets you design the network layout, define public/private split, and control routing precisely.

**Q5: Why should a database like RDS live in a private subnet?**
> To prevent it from being directly reachable from the internet, reducing attack surface. Only your application tier (in the public subnet) should be able to reach the database — enforced by subnet placement and Security Group rules together.

**Q6: What does /16 mean in a CIDR block like 10.0.0.0/16?**
> The first 16 bits are fixed (defining the network), and the remaining 16 bits are flexible for host addresses — giving 65,536 possible IP addresses in that range.

---

## 10. Hands-On Assignment for Today

1. Go to **AWS Console → VPC → Your VPCs** — examine the Default VPC.
   - Note its CIDR block (172.31.0.0/16)
   - Note how many subnets it has (one per AZ in your region)

2. Go to **VPC → Subnets** — look at the existing subnets:
   - Which AZ is each in?
   - What are their CIDR blocks?

3. **Create a custom VPC** (don't put resources in it yet — just create the structure):
   ```
   VPC → Create VPC
     Name: my-production-vpc
     CIDR: 10.0.0.0/16
   ```

4. **Create 2 public subnets** inside it:
   ```
   VPC → Subnets → Create subnet
     VPC: my-production-vpc
     Name: public-subnet-1a,  CIDR: 10.0.1.0/24,  AZ: ap-south-1a
     Name: public-subnet-1b,  CIDR: 10.0.2.0/24,  AZ: ap-south-1b
   ```

5. **Create 2 private subnets**:
   ```
     Name: private-subnet-1a, CIDR: 10.0.3.0/24,  AZ: ap-south-1a
     Name: private-subnet-1b, CIDR: 10.0.4.0/24,  AZ: ap-south-1b
   ```

6. Write a short note in your own words covering:
   - What a VPC is and why it exists
   - What makes a subnet public vs private
   - Why you created subnets in two AZs

---

## 11. Day 15 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"A VPC is my private network inside AWS, isolated from all other customers. I divide it into subnets — public subnets for internet-facing resources like EC2 and Load Balancers, and private subnets for databases like RDS that should never be directly reachable from the internet. Each subnet is pinned to one Availability Zone, so creating multi-AZ resources means creating subnets in multiple AZs. What actually makes a subnet public is its Route Table routing through an Internet Gateway — not a setting on the subnet itself."*

---

**Next up — Day 16: Internet Gateway & Route Tables**
