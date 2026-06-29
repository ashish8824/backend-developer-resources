# Day 17: Public vs Private Subnets
### *(Proving the Difference With Real Resources — The Most Important Network Concept in AWS)*

> **Roadmap reference:** Week 3, Day 17 — "Public vs Private Subnets"

---

## Why This Matters

Days 15 and 16 built the theory and the infrastructure. Today is the **hands-on proof** — you'll deploy real resources into both subnet types and observe exactly what the difference means in practice. This is the day the VPC concept stops being abstract and becomes something you can see, test, and explain with confidence.

Every production AWS architecture interview question about networking eventually comes back to this single concept: public subnet vs private subnet, and why the split exists.

---

## 1. Quick Recap — The Core Distinction

```
PUBLIC SUBNET                        PRIVATE SUBNET
─────────────                        ──────────────
Route Table has:                     Route Table has:
  10.0.0.0/16 → local          vs      10.0.0.0/16 → local
  0.0.0.0/0   → IGW                    (no internet route)

Resources CAN:                       Resources CAN:
  ✓ receive inbound internet traffic    ✗ receive inbound internet traffic
  ✓ initiate outbound internet traffic  ✗ initiate outbound internet traffic
                                        (without a NAT Gateway — Day 18)

Typical residents:                   Typical residents:
  EC2 (web/app servers)               RDS databases
  Load Balancers                       ElastiCache
  NAT Gateways                         Internal microservices
  Bastion hosts                         Backend processing workers
```

---

## 2. Today's Hands-On Plan

```
You will:

  1. Launch an EC2 instance in the PUBLIC subnet
     → Confirm it has internet access (SSH in, curl the web)

  2. Launch an EC2 instance in the PRIVATE subnet
     → Confirm it has NO internet access
     → Confirm it CAN be reached FROM the public EC2
       (internal VPC communication works fine)

  3. Understand the Bastion Host pattern
     → How engineers safely access private resources

  4. Clean up (terminate instances)
```

---

## 3. Step 1 — EC2 in the Public Subnet

```
EC2 → Launch Instance

  Name: public-ec2-test
  AMI: Ubuntu Server 22.04 LTS
  Instance type: t2.micro

  Network settings:
    VPC: my-production-vpc
    Subnet: public-subnet-1a
    Auto-assign public IP: ENABLE ← critical
    Security Group: allow SSH from My IP, HTTP from anywhere

  → Launch
```

### Verify Internet Access

```bash
# SSH in using its public IP
ssh -i my-key.pem ubuntu@<public-ec2-public-ip>

# Test outbound internet
curl https://api.ipify.org
# → Returns a public IP address — confirms internet access ✓

# Test DNS resolution works
ping -c 3 google.com
# → Should get responses ✓
```

---

## 4. Step 2 — EC2 in the Private Subnet

```
EC2 → Launch Instance

  Name: private-ec2-test
  AMI: Ubuntu Server 22.04 LTS
  Instance type: t2.micro

  Network settings:
    VPC: my-production-vpc
    Subnet: private-subnet-1a
    Auto-assign public IP: DISABLE ← no public IP possible
    Security Group: allow SSH from the PUBLIC subnet's CIDR
                   (10.0.1.0/24) — not from your laptop directly

  → Launch
```

```
Notice: The private EC2 gets only a PRIVATE IP (e.g., 10.0.3.45)
        There is no public IP column — it's blank.
        You CANNOT SSH into this instance directly from your laptop.
```

---

## 5. The Bastion Host Pattern — How to Access Private Resources

**A Bastion Host (also called a Jump Server) is a hardened EC2 instance in a public subnet that acts as a secure entry point to reach resources in private subnets.**

```
Your Laptop
    │
    │  SSH (port 22)
    ▼
Public EC2 (Bastion Host)     ← in public subnet, has public IP
    │
    │  SSH (port 22) — internal VPC traffic only
    ▼
Private EC2 / RDS              ← in private subnet, no public IP
```

```
Why this pattern?
  Your private resources have no public IP — unreachable from internet.
  The bastion is the ONLY entry point, and you harden it:
    - SSH restricted to your IP only
    - Minimal software installed
    - Monitored heavily
    - Often requires MFA

  Even if an attacker somehow found your private EC2, they'd
  need to compromise the bastion first — an additional barrier.
```

### SSH from Public EC2 into Private EC2

```bash
# On your LOCAL machine — copy your private key to the public EC2
# (so you can use it to SSH further into the private EC2)
scp -i my-key.pem my-key.pem ubuntu@<public-ec2-ip>:~/.ssh/

# SSH into the public EC2 (bastion)
ssh -i my-key.pem ubuntu@<public-ec2-ip>

# FROM the public EC2, SSH into the private EC2
ssh -i ~/.ssh/my-key.pem ubuntu@<private-ec2-private-ip>
# e.g.: ssh -i ~/.ssh/my-key.pem ubuntu@10.0.3.45
```

```
Once inside the private EC2, test internet access:

curl https://api.ipify.org
# → Connection timed out — NO internet access ✓

ping google.com
# → Network unreachable or timeout ✓

# BUT — internal VPC traffic works fine:
ping <public-ec2-private-ip>
# → Gets responses — VPC internal routing works ✓
```

> This proves precisely what "private subnet" means: **not reachable from the internet, not able to reach the internet** — but can communicate freely with other resources inside the same VPC.

---

## 6. The Two-Tier Architecture — Proven in Practice

```
                    Internet
                        │
                        │ Your SSH
                        ▼
          ┌─────────────────────────────┐
          │  PUBLIC SUBNET (10.0.1.0/24)  │
          │                               │
          │   public-ec2-test              │  ← has public IP
          │   (Bastion Host)               │  ← reachable from internet
          │   Private IP: 10.0.1.10         │  ← can reach internet
          └─────────────────────────────┘
                        │
                        │ Internal VPC traffic (fast, free)
                        ▼
          ┌─────────────────────────────┐
          │  PRIVATE SUBNET (10.0.3.0/24) │
          │                               │
          │   private-ec2-test             │  ← NO public IP
          │   Private IP: 10.0.3.45         │  ← NOT reachable from internet
          │   (Future: RDS would go here)   │  ← CAN'T reach internet
          └─────────────────────────────┘
```

---

## 7. Auto-Assign Public IP — An Important Detail

```
When you launch EC2 into a subnet, AWS can optionally
auto-assign a public IP. This setting has two levels:

SUBNET-LEVEL default setting:
  → Each subnet has a default "auto-assign public IPv4" setting
  → Public subnets: usually set to ENABLED by default
  → Private subnets: should be set to DISABLED

INSTANCE-LEVEL override:
  → During launch, you can override the subnet's default
  → Enabling on a private subnet instance won't help though —
    a public IP without an IGW route is still unreachable

Best practice:
  → Set public subnets: auto-assign = ENABLED
  → Set private subnets: auto-assign = DISABLED
  → This prevents accidentally giving public IPs to private resources
```

```
AWS Console → VPC → Subnets → select public-subnet-1a
  → Actions → Edit subnet settings
  → Enable auto-assign public IPv4 address ✓
  → Save

Do the SAME for public-subnet-1b.

For private subnets — confirm this is DISABLED (default is off).
```

---

## 8. Security Group Layering With Subnets

Now that you understand both layers, here's how they work in combination:

```
Traffic from internet targeting your private EC2:

Step 1: Route Table check
  Private subnet Route Table: no 0.0.0.0/0 → IGW route
  → BLOCKED at network level. Traffic never even reaches the instance.
  (Security Group never gets evaluated)

Traffic from internet targeting your public EC2:

Step 1: Route Table check
  Public subnet Route Table: 0.0.0.0/0 → IGW
  → ALLOWED to reach the subnet ✓

Step 2: Security Group check
  Does the Security Group allow this port/IP?
  → If YES: traffic reaches the instance ✓
  → If NO: BLOCKED at instance level ✗
```

```
The layered model:
  Subnet/Route Table = "Does a road to this destination exist?"
  Security Group     = "Is this specific car allowed to park here?"

Both gates must be open for traffic to get through.
```

---

## 9. Real-World Architecture — Three Tiers

In larger production systems, you'll often see a **three-tier architecture** built on this same concept:

```
                      Internet
                          │
              ┌───────────▼────────────┐
              │      PUBLIC SUBNET       │
              │   [Load Balancer]         │  ← receives all public traffic
              └───────────┬────────────┘
                          │ internal only
              ┌───────────▼────────────┐
              │    APP/PRIVATE SUBNET    │
              │   [EC2 / ECS Fargate]    │  ← runs your actual code
              │   [Express / Node.js]     │  ← NOT directly internet-facing
              └───────────┬────────────┘
                          │ internal only
              ┌───────────▼────────────┐
              │      DATA SUBNET         │
              │   [RDS PostgreSQL]        │  ← data layer, most restricted
              │   [ElastiCache Redis]     │  ← never internet-reachable
              └────────────────────────┘
```

> This is exactly the pattern used for a production version of PulseBloom or QueueCare — Load Balancer in public subnet, ECS Fargate tasks in a private app subnet, RDS in a private data subnet.

---

## 10. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is a Bastion Host, and why is it used?**
> A hardened EC2 instance in a public subnet that acts as the sole authorized entry point to reach resources in private subnets. Since private resources have no public IP, the bastion is the only SSH path — centralizing and controlling admin access.

**Q2: If an EC2 instance is in a private subnet with no public IP, can it ever reach the internet?**
> Not by default. Adding a NAT Gateway (in the public subnet) and updating the private Route Table to route `0.0.0.0/0` through it allows outbound internet access without allowing inbound connections — covered on Day 18.

**Q3: Can two EC2 instances in different subnets of the same VPC communicate with each other?**
> Yes — the local route (`10.0.0.0/16 → local`) in every Route Table covers internal VPC communication. An EC2 in the public subnet can reach an EC2 or RDS in the private subnet via their private IPs, as long as Security Groups permit it.

**Q4: What happens if you assign a public IP to an EC2 instance in a private subnet?**
> The instance gets a public IP, but it's still unreachable from and can't reach the internet — because the private subnet's Route Table has no `0.0.0.0/0 → IGW` route. The public IP is effectively useless in a private subnet without the corresponding route.

**Q5: In a production three-tier architecture, which tier should the Load Balancer be in?**
> The public subnet — the Load Balancer is the only internet-facing component. The application servers behind it should be in private subnets, receiving traffic only from the Load Balancer's Security Group, not directly from the internet.

**Q6: Why is placing application servers in private subnets (even behind a Load Balancer) better than putting them in public subnets?**
> It reduces attack surface — even if the Load Balancer is compromised or misconfigured, the application servers aren't directly reachable from the internet. Attackers would need to compromise the Load Balancer first to even attempt reaching the app tier.

---

## 11. Hands-On Assignment for Today

1. **Launch public-ec2-test** in `public-subnet-1a` with auto-assign public IP enabled — SSH in and confirm `curl https://api.ipify.org` works.

2. **Launch private-ec2-test** in `private-subnet-1a` with no public IP — confirm it gets only a private IP.

3. **SSH from public EC2 into private EC2** using the bastion pattern — confirm you can reach it internally.

4. **Run these tests from inside the private EC2** and document the results:
   ```bash
   curl https://api.ipify.org         # should timeout ✗
   ping google.com                     # should timeout ✗
   ping <public-ec2-private-ip>        # should work ✓
   ```

5. **Enable auto-assign public IP** on both public subnets at the subnet settings level.

6. **Terminate both test instances** when done — you don't need them running.

7. Write a short note covering:
   - Why the private EC2 could ping the public EC2 but not Google
   - What a Bastion Host is and when you'd use it
   - How the three-tier architecture maps to the services you've already built

---

## 12. Day 17 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"A public subnet has a Route Table entry pointing to an Internet Gateway, making resources there reachable from and able to reach the internet. A private subnet has no such route — resources there are completely isolated from the internet but can still communicate with other VPC resources internally. To access private resources, engineers use a Bastion Host in the public subnet as a jump server. In a real production app, only the Load Balancer lives in the public subnet — app servers and databases live in private subnets for maximum security."*

---

**Next up — Day 18: Load Balancer (ALB)**
