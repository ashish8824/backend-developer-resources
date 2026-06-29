# Day 1: Cloud Computing Fundamentals + AWS Global Infrastructure
### *(Why Cloud Exists, Where AWS Lives, and Your First AWS Account)*

> **Roadmap reference:** Week 1, Day 1 — "What is Cloud Computing? | AWS Global Infrastructure | Regions & Availability Zones | Create AWS Free Tier Account"

---

## Why Start Here?

Before you touch a single AWS service, you need to answer three questions confidently:

```
1. Why does cloud computing exist?
   (What problem does it solve over owning physical servers?)

2. Where does AWS physically live?
   (What are Regions, AZs, and Edge Locations?)

3. How do I get started safely?
   (Free Tier account with cost guardrails from day one)
```

If you can explain these three things clearly to a non-technical person, you've cleared the first checkpoint that most interviewers use to test whether someone truly understands AWS — or just knows how to click around the console.

---

## SECTION A: What is Cloud Computing?

---

## 1. The Old Way vs The Cloud Way

**Cloud computing means renting computing resources — servers, storage, databases, networking — over the internet, instead of buying and owning them yourself.**

### The Old Way (On-Premise)

```
Buy Server → Set Up Data Center → Install OS → Configure Network →
Deploy Application → Maintain Hardware → Replace When It Fails
```

- You own everything: the physical machine, the rack, the cooling, the power backup, the security guards.
- Upfront cost: ₹50,000+ just for a basic server, before writing a single line of code.
- If the hard drive fails at 3 AM — that's your problem.

### The Cloud Way

```
Create EC2 Instance → Deploy Application
```

- AWS owns the hardware, data centers, networking, and physical security.
- You **use** the resources and pay only for what you consume.
- If hardware fails — AWS handles it, often before you even notice.

### Side-by-Side

| | On-Premise | Cloud (AWS) |
|---|---|---|
| Setup time | Weeks | Minutes |
| Upfront cost | High (buy hardware) | Near-zero |
| Scaling | Buy more physical machines | Click a button |
| Maintenance | You handle everything | AWS handles hardware layer |
| Payment model | Buy once, own forever | Pay-as-you-go |

---

## 2. The Simple Analogy — Electricity

> **Cloud computing is to servers what electricity is to power.**

You don't build a power plant in your backyard to run your fridge. You plug into the grid and pay for what you consume. AWS owns the "power plant" (data centers) and you just plug in.

---

## 3. The Three Real Problems Cloud Solves

### Problem 1: Wasted Money on Guessing Capacity

```
Website launch
     ↓
Company buys 100 servers (expecting big traffic)
     ↓
Only 10 users show up on launch day
     ↓
90 servers sit idle — money wasted, no refunds
```

### Problem 2: Scaling Too Slow

```
Big Sale Day → traffic spikes 50x overnight
     ↓
Need more servers urgently
     ↓
Order hardware → wait weeks for delivery
     ↓
By the time servers arrive, the sale is over
```

### Problem 3: Maintenance Overhead

Without cloud, your team must manage:
- Hardware failures and replacements
- Power backup systems
- Cooling systems
- Physical security guards for the server room
- Network cable management

Cloud absorbs all of this so your team focuses on the product.

---

## 4. Service Models — IaaS, PaaS, SaaS

**The most frequently tested cloud concept in interviews.** The core idea: as you move from IaaS → PaaS → SaaS, AWS manages more and you manage less.

```
IaaS                PaaS               SaaS
─────               ─────              ─────
You manage:         You manage:        You manage:
 OS                  Your code only     Nothing
 Runtime
 Application

AWS manages:        AWS manages:       AWS manages:
 Hardware            Hardware           Everything
 Networking          OS
 Storage             Runtime
```

### IaaS — Infrastructure as a Service
AWS gives you raw building blocks. You install and configure everything on top.
- **Example:** Amazon EC2
- *Analogy: Renting an empty apartment — you bring your own furniture.*

### PaaS — Platform as a Service
AWS manages the server, OS, and runtime. You only deploy your code.
- **Example:** AWS Elastic Beanstalk
- *Analogy: Renting a furnished apartment — you just move in.*

### SaaS — Software as a Service
Everything managed. You just use the finished product.
- **Examples:** Gmail, Google Docs, Microsoft 365
- *Analogy: Staying in a hotel — everything is ready.*

| Model | AWS Manages | You Manage | Example |
|---|---|---|---|
| IaaS | Hardware, networking, storage | OS, runtime, app, data | Amazon EC2 |
| PaaS | Hardware, OS, runtime | Just your application code | Elastic Beanstalk |
| SaaS | Everything | Nothing — just use it | Gmail |

---

## 5. The 5 Defining Characteristics of Cloud Computing

These are the official NIST definition — memorize them for interviews:

```
1. On-Demand Self-Service
   Create what you need, when you need it — no human approval.
   Example: Launch an EC2 server in 2 minutes, by yourself.

2. Pay-As-You-Go
   Billed only for what you actually consume.
   Example: Run a server for 5 hours → pay for 5 hours.

3. Elasticity (Scalability)
   Resources grow or shrink automatically with demand.
   100 users → 100,000 users → AWS scales automatically.

4. Global Availability
   Deploy in multiple countries within minutes using
   AWS's global data centers.

5. High Availability
   If one server fails, a backup takes over — application
   stays online. Users never notice.
```

---

## SECTION B: AWS Global Infrastructure

---

## 6. The Three Layers of AWS Infrastructure

AWS infrastructure is organized as nested layers, from largest to smallest:

```
            🌍 AWS GLOBAL INFRASTRUCTURE
                        │
        ┌───────────────┼───────────────┐
        │               │               │
     REGION          REGION          REGION
   (Mumbai)         (Virginia)      (Singapore)
        │
        ├── Availability Zone A
        ├── Availability Zone B   ← fast fiber links between AZs
        └── Availability Zone C

        │
   Edge Locations (hundreds of cities worldwide — for CloudFront/Route 53)
```

Think of it like a company's structure:
- **Region** = a country office (e.g., the India office)
- **Availability Zone** = a specific building within that campus
- **Edge Location** = a local kiosk near the customer, just for fast content delivery

---

## 7. AWS Regions

**A Region is a physical geographic location where AWS has clustered a group of independent, isolated data centers.**

Each region is completely independent — a failure in one region does not affect others.

### Real AWS Regions

| Region Code | Location |
|---|---|
| `ap-south-1` | Mumbai, India |
| `us-east-1` | N. Virginia, USA |
| `eu-west-1` | Ireland |
| `ap-southeast-1` | Singapore |

> AWS operates in 30+ Regions worldwide, with more being added regularly.

### How to Choose a Region?

```
Three factors decide it almost every time:

1. Latency       → Pick the region closest to your majority users
                   Indian users? → ap-south-1 (Mumbai)

2. Compliance    → Some data must legally stay within a country's borders
                   e.g., EU healthcare data must stay in EU

3. Service       → Not every AWS service is available in every region
   Availability    on day one — check for the specific service you need
```

---

## 8. Availability Zones (AZs)

**An Availability Zone is one or more physically separate data centers within a Region, each with its own independent power, cooling, and networking.**

Every Region has **3 or more AZs** — so if one data center has a power outage, flood, or hardware failure, your application keeps running from another AZ.

```
Region: ap-south-1 (Mumbai)
┌─────────────────────────────────────────────┐
│                                               │
│   AZ: ap-south-1a   AZ: ap-south-1b   AZ: ap-south-1c │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐  │
│   │ Data     │◄────►│ Data     │◄────►│ Data     │  │
│   │ Center   │ fast │ Center   │ fast │ Center   │  │
│   │          │fiber │          │fiber │          │  │
│   └──────────┘      └──────────┘      └──────────┘  │
│                                                        │
└─────────────────────────────────────────────┘
```

- AZs within the same region are connected by **high-speed, low-latency private fiber** — data replicates between them almost instantly.
- This is the technical foundation of **High Availability**.

### Why Multiple AZs Matter — Real Example

```
Your Node.js + PostgreSQL app runs in ONE AZ:
  AZ goes down (power failure) → entire app offline → all users see errors

Your app deployed across MULTIPLE AZs with Load Balancer:
  AZ goes down → Load Balancer detects → routes to other AZs
  Users notice nothing → zero downtime
```

---

## 9. Edge Locations

**An Edge Location is a small AWS data center, spread across hundreds of cities, used to cache and deliver content as close to the end user as physically possible.**

Edge Locations don't run EC2 or RDS — their only job is **speed up content delivery** via:
- **CloudFront** (AWS's Content Delivery Network)
- **Route 53** (DNS resolution)

### Why Edge Locations Exist

```
Without Edge Locations:
  User in Mumbai requests an image from your server in us-east-1
  → Request travels ~14,000 km round trip → ~250ms delay

With Edge Locations:
  Same user's request served from Mumbai Edge Location
  → ~10km round trip → ~5ms — feels instant
```

> There are **600+ Edge Locations** globally vs 30+ Regions — caching needs to be hyper-local to be effective.

---

## 10. Full Infrastructure Summary

| Layer | What It Is | Used For | Count |
|---|---|---|---|
| **Region** | Geographic cluster of data centers | Where your app/data lives | 30+ |
| **Availability Zone** | Isolated data center(s) in a Region | Fault tolerance & HA | 2–6 per Region |
| **Edge Location** | Small local caching node | CloudFront fast delivery | 600+ |

---

## SECTION C: Create Your AWS Free Tier Account

---

## 11. What is AWS Free Tier?

**AWS Free Tier gives new accounts free usage of many services so you can learn without paying — but you must understand its limits to avoid surprise bills.**

Three types of free offers:

```
ALWAYS FREE
  → Free forever for every AWS account, within monthly limits
  → Example: Lambda — 1 million requests/month free, always

12 MONTHS FREE
  → Only free during your first 12 months after account creation
  → After 12 months: normal pricing kicks in automatically
  → Example: EC2 t2.micro/t3.micro — 750 hours/month

TRIALS
  → Short-term free period after activating a specific service
  → Example: Some database or ML services offer 30-day trials
```

---

## 12. Creating Your AWS Free Tier Account

```
Step 1: Go to https://aws.amazon.com → "Create an AWS Account"

Step 2: Enter your email address + account name
  → Choose a strong, unique password
  → This email becomes your Root account — protect it carefully

Step 3: Account type → Personal (for learning)

Step 4: Contact information → fill in your details

Step 5: Billing information → credit/debit card required
  → AWS uses it to verify identity and for any charges
  → You will NOT be charged if you stay within Free Tier limits
  → A small $1 temporary hold may appear for verification

Step 6: Identity verification → enter the code sent to your phone

Step 7: Support plan → choose "Basic support" (FREE)
  → Do NOT select Developer or Business — those cost money

Step 8: Sign in to the AWS Console
  → Go to: https://console.aws.amazon.com
  → Sign in as Root user with your email + password
```

---

## 13. Essential Cost Safety Setup (Do This IMMEDIATELY)

Before exploring anything else, set up these four guardrails:

### Guardrail 1: Set a Billing Alert

```
AWS Console → Billing Dashboard → Budgets → Create budget
  Budget type: Cost budget
  Period: Monthly
  Budget amount: $5 (or $1 — anything that alerts you early)
  Alert: 80% of budget → email yourself
  → Create budget
```

### Guardrail 2: Enable Free Tier Alerts

```
Billing Dashboard → Billing preferences
  → Check: "Receive Free Tier Usage Alerts"
  → Enter your email
  → Save preferences
```

### Guardrail 3: Create an IAM User (Never Use Root Daily)

```
IAM → Users → Create user
  Username: your-name-dev
  → Attach policy: AdministratorAccess (for learning)
  → Download credentials or set a console password
  → Use THIS user for all daily work, never Root
```

### Guardrail 4: Bookmark the Billing Dashboard

```
AWS Console → Billing Dashboard → Bookmark this page
Check it weekly — 2 minutes is enough to catch a forgotten resource
```

---

## 14. First Exploration (Hands-On)

Once your account is ready, explore — don't create anything yet:

```
1. Top-right corner: click the Region dropdown
   → Browse all available regions
   → Switch to ap-south-1 (Mumbai) — use this for all practice

2. Click "Services" in the top navigation
   → Find and click on: EC2, S3, IAM, RDS
   → Read what each service does (hover over descriptions)

3. Go to IAM:
   → Notice the "Security recommendations" warning about Root
   → This is why you created the IAM user above

4. Go to EC2:
   → Click "Launch Instance" — just browse the options, don't submit
   → Look at AMIs, Instance Types, Key Pair options
   → This is what you'll actually create on Day 4
```

---

## 15. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is Cloud Computing?**
> The delivery of computing resources — servers, storage, databases, networking — over the internet on a pay-as-you-go basis, instead of owning and managing physical infrastructure.

**Q2: What are the three cloud service models?**
> IaaS (Infrastructure as a Service — e.g., EC2), PaaS (Platform as a Service — e.g., Elastic Beanstalk), and SaaS (Software as a Service — e.g., Gmail). As you go from IaaS to SaaS, AWS manages more and you manage less.

**Q3: What is an AWS Region?**
> A physical geographic location where AWS has a cluster of independent, isolated data centers. Each region is completely independent, so failures don't cross between regions.

**Q4: What is an Availability Zone?**
> One or more physically separate data centers within a Region, each with its own power, cooling, and networking. Multiple AZs in a region provide fault tolerance — if one goes down, others keep your application running.

**Q5: What is the difference between an Availability Zone and an Edge Location?**
> An AZ is a full data center that runs EC2, RDS, and other services. An Edge Location is a smaller caching node in cities worldwide used only for content delivery (CloudFront) and DNS (Route 53).

**Q6: What are the three types of AWS Free Tier offers?**
> Always Free (forever, every account, with limits), 12 Months Free (first year only after account creation), and Trials (short-term after activating a specific service).

---

## 16. Day 1 Goal — Can You Teach This?

By the end of Day 1, you should be able to explain, without notes:

> *"Cloud computing means renting servers, storage, and databases over the internet instead of owning them — paying only for what you use, scaling instantly, and letting AWS manage the physical hardware. AWS infrastructure is organized as Regions (geographic locations), which contain multiple Availability Zones (isolated data centers for fault tolerance), supported by Edge Locations worldwide for fast content delivery. I've created a Free Tier account with billing alerts and an IAM user so I never touch Root credentials for daily work."*

---

**Next up — Day 2: AWS Console Navigation, Billing Dashboard & Shared Responsibility Model**
