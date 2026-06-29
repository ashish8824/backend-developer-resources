# Day 2: AWS Console Navigation + Billing Dashboard + Shared Responsibility Model
### *(Learning the Interface, Protecting Your Wallet, and Understanding Security Ownership)*

> **Roadmap reference:** Week 1, Day 2 — "AWS Console Navigation | Billing Dashboard | Shared Responsibility Model"

---

## Why This Matters

Day 1 created your account. Day 2 teaches you how to USE it safely. Three things you absolutely must master before touching any real AWS service:

```
1. Console Navigation   → know WHERE everything is, fast
2. Billing Dashboard    → know HOW MUCH things cost, always
3. Shared Responsibility → know WHAT YOU are responsible for securing
```

Miss any of these and you risk being lost in the UI, surprised by unexpected bills, or — most dangerously — thinking AWS secures everything for you when it absolutely doesn't.

---

## SECTION A: AWS Console Navigation

---

## 1. The AWS Management Console — Layout Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  ← AWS Logo  |  Services ▼  |  Search bar  |  Region ▼  |  Account ▼ │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Recently visited:                                                 │
│  [EC2]  [S3]  [IAM]  [RDS]  [Lambda]                            │
│                                                                    │
│  AWS Health  |  Cost & Usage  |  Featured services                │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### The 4 Most Important Console Elements

```
1. SERVICES MENU (top left)
   → Dropdown with ALL AWS services organized by category
   → Categories: Compute, Storage, Database, Networking,
                 Security, Serverless, Machine Learning, etc.
   → Quick access to any service in seconds

2. SEARCH BAR (center top)
   → Fastest way to navigate — just type the service name
   → Try: type "EC2" → press Enter → lands on EC2 dashboard
   → Also searches documentation, blog posts, marketplace

3. REGION SELECTOR (top right, next to your account name)
   → Shows your current region (e.g., "Asia Pacific (Mumbai)")
   → CRITICAL: Every resource you create belongs to ONE region
   → If you create an EC2 in Mumbai and switch to Virginia,
     you won't see it — it's in Mumbai, not Virginia
   → Always check your region before creating anything

4. ACCOUNT MENU (top right, your account name)
   → Account settings, billing, security credentials
   → Sign out
   → Switch roles (for multi-account setups)
```

---

## 2. The Region Selector — The #1 Source of Beginner Confusion

```
SYMPTOM: "I created an EC2 instance but I can't find it!"
CAUSE:   You were in Mumbai when you created it, but you're
         now looking in Virginia.

FIX: ALWAYS check the region selector before:
  → Creating any resource
  → Looking for an existing resource
  → Checking a bill (cost by region)
```

```
Setting your default working region:
  → Top-right → click current region name
  → Select: Asia Pacific (Mumbai) ap-south-1
  → This is the natural default for Indian projects (low latency)

The region selector location:
  Top navigation bar → right side → looks like: "Asia Pacific (Mumbai) ▼"
```

---

## 3. Pinning Your Most-Used Services

AWS has 200+ services. You'll use maybe 15 regularly. Pin them:

```
Console Home → Edit (pencil icon near "Recently visited")
  → Search and add:
    ✓ EC2
    ✓ S3
    ✓ IAM
    ✓ RDS
    ✓ Lambda
    ✓ VPC
    ✓ CloudWatch
    ✓ ECS
    ✓ Billing

These now appear on your console home page for one-click access.
```

---

## 4. Key Service Dashboards — Know These Layouts

### EC2 Dashboard

```
EC2 → left sidebar:
  Instances        → your running virtual servers
  Images (AMIs)    → server templates
  Security Groups  → firewall rules
  Key Pairs        → SSH login credentials
  Elastic IPs      → static public IP addresses
  Load Balancers   → traffic distribution
  Auto Scaling     → automatic instance management
  Volumes (EBS)    → storage disks
```

### S3 Dashboard

```
S3 → simple layout:
  All Buckets      → list of all your storage containers
  (click bucket)
    Objects        → files inside the bucket
    Permissions    → access control settings
    Properties     → versioning, logging, encryption
```

### IAM Dashboard

```
IAM → left sidebar:
  Users           → individual identities
  User Groups     → collections of users
  Roles           → temporary identities for services
  Policies        → JSON permission documents
  Security Status  → root account MFA, etc.
```

---

## 5. Using the Search Bar Effectively

```
Type service name → instant access:
  "ec2"      → EC2 dashboard
  "s3"       → S3 dashboard
  "iam"      → IAM dashboard
  "rds"      → RDS dashboard
  "lambda"   → Lambda dashboard
  "vpc"      → VPC dashboard

Search also finds:
  "create bucket"     → takes you directly to S3 create bucket page
  "launch instance"   → takes you directly to EC2 launch wizard
  "create user"       → takes you directly to IAM create user page

This is the fastest way to navigate — don't waste time clicking
through menus when you can type the destination directly.
```

---

## 6. CloudShell — Your Browser-Based Terminal

```
Top navigation bar → CloudShell icon (terminal icon, top right)

→ Opens a browser-based terminal pre-authenticated with your
  AWS credentials
→ AWS CLI pre-installed — no setup needed
→ Great for quick commands without SSH into an EC2

Try it:
  aws s3 ls              # list all your S3 buckets
  aws ec2 describe-instances  # list EC2 instances
  aws iam list-users         # list IAM users
```

---

## SECTION B: Billing Dashboard

---

## 7. Finding the Billing Dashboard

```
Method 1: Account menu (top right) → "Billing and Cost Management"
Method 2: Search bar → type "billing"
Method 3: Services → Management & Governance → Billing

Bookmark it immediately — you should check this weekly.
```

---

## 8. The Billing Dashboard — What Each Section Tells You

```
┌─────────────────────────────────────────────────────────────┐
│  BILLING DASHBOARD                                            │
├──────────────────┬──────────────────────────────────────────┤
│  Bills            │ Exact breakdown of what you're being     │
│                   │ charged, per service, per region          │
│                   │ Updated daily                            │
├──────────────────┼──────────────────────────────────────────┤
│  Cost Explorer    │ Visual charts of your spending over time  │
│                   │ Filter by service, region, time period    │
│                   │ "Why did my bill go up this week?"        │
├──────────────────┼──────────────────────────────────────────┤
│  Budgets          │ Set spending limits + automatic alerts    │
│                   │ Email/SMS when you approach or hit limit  │
├──────────────────┼──────────────────────────────────────────┤
│  Free Tier        │ Tracks how close you are to each         │
│                   │ free tier service limit this month        │
│                   │ "EC2: 523/750 hours used this month"     │
├──────────────────┼──────────────────────────────────────────┤
│  Cost Allocation  │ Tag resources to track costs by project  │
│  Tags             │ e.g., tag all PulseBloom resources →     │
│                   │ see PulseBloom's total cost separately    │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 9. How AWS Billing Actually Works

```
Your Monthly Bill = 
    EC2 hours used      × hourly rate
  + S3 storage (GB)     × per-GB rate
  + Data transfer OUT   × per-GB rate
  + RDS hours used      × hourly rate
  + Lambda invocations  × per-million rate
  + ... (every service you used)
```

### The Most Important Hidden Rule: Data Transfer

```
Data IN to AWS (uploads, downloads FROM internet to AWS):
  → Usually FREE

Data OUT of AWS (serving data TO users from AWS):
  → CHARGED per GB

Example:
  You upload 10 GB of images to S3   → FREE
  1,000 users download those images  → CHARGED (data transfer out)

This is the most common "invisible" cost beginners discover late.
```

---

## 10. Setting Up a Budget Alert (Do This Now If You Didn't Yesterday)

```
Billing Dashboard → Budgets → Create budget

  Step 1: Budget type → Cost budget → Next

  Step 2: Set budget amount:
    Period: Monthly
    Budget amount: $5
    (AWS Free Tier is mostly free but $5 alert
     will catch anything slipping through)

  Step 3: Set alert:
    Alert threshold: 80% of budgeted amount ($4)
    Email: your@email.com
    → Add notification

  Step 4: Review → Create budget
```

```
Also enable Free Tier alerts:
  Billing Dashboard → Billing preferences (left sidebar)
    → Check: "Receive Free Tier Usage Alerts"
    → Email: your@email.com
    → Save preferences
```

---

## 11. The Bills Page — Reading Your First Bill

```
Billing → Bills → select current month

View by:
  SERVICE → shows cost broken down by EC2, S3, RDS, etc.
  REGION  → shows which region generated which cost

Zero-cost month (expected for new Free Tier accounts):
  All items should show $0.00 — good, that means Free Tier is working.

If you see an unexpected charge:
  → Click the service to expand and see exact usage
  → Common culprits: forgotten running EC2, data transfer,
    Elastic IPs not attached to running instances
```

---

## 12. Cost Explorer — Visualizing Spending

```
Billing → Cost Explorer → Launch Cost Explorer

Time range: Last 6 months
Group by: Service

This shows a bar chart of your spending over time, broken by service.
Even with $0 spending, explore the interface — you'll use this when
real costs appear later in the course.

Filter options:
  By Service: "Show me only EC2 costs"
  By Region:  "Show me only Mumbai region costs"
  By Tag:     "Show me all costs tagged 'project: pulsebloom'"
```

---

## 13. Free Tier Usage Page — Your Early Warning System

```
Billing → Free Tier (left sidebar)

Shows all services with free limits and your current usage:

  Service          | Free Limit      | Used This Month | % Used
  ─────────────────────────────────────────────────────────────
  EC2 t3.micro     | 750 hours       | 0 hours         | 0%
  S3 Storage       | 5 GB            | 0 GB            | 0%
  RDS t3.micro     | 750 hours       | 0 hours         | 0%
  Lambda requests  | 1M requests     | 0               | 0%
  CloudWatch       | 10 custom metrics | 0             | 0%

AWS emails you when any service hits 85% of its free limit
(if you enabled Free Tier alerts above).
```

---

## SECTION C: Shared Responsibility Model

---

## 14. The Most Misunderstood Security Concept in AWS

The single most dangerous misconception beginners have:

> ❌ *"AWS hosts my app, so AWS keeps it secure."*

**This is false. Believing this is exactly how real-world data breaches happen.**

AWS secures the cloud itself — but a huge portion of security remains 100% your responsibility.

---

## 15. The Shared Responsibility Model

**The Shared Responsibility Model splits security between AWS and the customer.**

The cleanest way to remember it — one sentence that interviewers use constantly:

```
AWS is responsible for security  OF  the cloud
You are responsible for security  IN  the cloud
```

One preposition separates the two halves. Memorize that line.

---

## 16. The Split — Visualized

```
┌──────────────────────────────────────────────────────────┐
│              YOUR RESPONSIBILITY ("IN the cloud")          │
│                                                             │
│   Your Application Code (no SQL injection, no vulnerabilities)│
│   Your Data (encryption, classification, access control)   │
│   IAM Users/Roles (who can do what — least privilege)       │
│   OS Patching on EC2 (you must update Ubuntu yourself)       │
│   Security Group Rules (which ports you open)                │
│   Network Configuration (VPC, subnets, NACLs)               │
│   Client-side & Server-side Encryption settings              │
├──────────────────────────────────────────────────────────┤
│              AWS'S RESPONSIBILITY ("OF the cloud")          │
│                                                             │
│   Physical Data Centers (guards, cameras, access control)   │
│   Physical Hardware (servers, racks, disks)                  │
│   Networking Infrastructure (global backbone)               │
│   Host OS / Hypervisor (what runs under your EC2)           │
│   Power, Cooling, Physical Security at facilities           │
└──────────────────────────────────────────────────────────┘
```

---

## 17. The Apartment Analogy

> Think of AWS as a landlord and your app as a tenant.

```
Landlord (AWS) guarantees:
  → Building is structurally sound
  → Main entrance locks are working
  → Electricity and plumbing work
  → Building security guards are present

Tenant (You) is responsible for:
  → Locking YOUR apartment door
  → Not leaving your windows open
  → Not giving your keys to strangers
  → Not storing dangerous materials

If you leave your apartment door unlocked and get robbed:
  → That's NOT the landlord's fault
  → The building was secure — YOUR door wasn't
```

---

## 18. The Responsibility Split Changes With the Service Type

This is a more nuanced point — excellent to mention in interviews:

```
IaaS (e.g., EC2) → YOU manage the most:
  → AWS: hardware, networking, hypervisor
  → You: OS patching, runtime, app code, data, network config

PaaS (e.g., Lambda, Elastic Beanstalk) → AWS manages more:
  → AWS: hardware, OS, runtime patching
  → You: your application code and data only

SaaS → AWS manages almost everything:
  → You: data you put in, access control settings
```

```
Practical example:

  EC2 (IaaS):
    → You must manually run: sudo apt update && sudo apt upgrade
    → AWS does NOT patch your Ubuntu OS for you
    → If you don't patch, a vulnerability in Ubuntu is YOUR problem

  Lambda (Serverless/PaaS):
    → AWS automatically patches the Node.js runtime
    → You only worry about your function code
    → Smaller attack surface = "more secure by default"
```

---

## 19. A Real Security Mistake — Made Concrete

```
Scenario: Developer is learning AWS for the first time.

Day 1: Launches EC2 instance → Security Group set to:
  Inbound: SSH (port 22) from 0.0.0.0/0 (ENTIRE internet)

What AWS secures:
  ✓ Physical data center is locked
  ✓ Hardware is protected
  ✓ Network backbone is secure

What YOU failed to secure:
  ✗ Security Group allows SSH from ANYONE on the internet
  ✗ Automated bots constantly scan public IPs for open port 22
  ✗ Within hours, brute-force attempts start hitting your server

Whose responsibility is this?
  → 100% YOURS. AWS gave you the Security Group tool correctly.
    Configuring it insecurely is your mistake, not AWS's.
```

---

## 20. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is the AWS Shared Responsibility Model?**
> A security framework defining what AWS secures (the physical infrastructure, hardware, and data center — "security OF the cloud") versus what the customer secures (data, IAM configuration, OS patches, network settings — "security IN the cloud").

**Q2: Give one example of something AWS is responsible for.**
> Physical data center security, hardware maintenance, and the networking infrastructure connecting Regions and AZs — the customer never sees or configures any of this.

**Q3: Give one example of something the customer is responsible for.**
> Configuring Security Group rules correctly, patching the OS on EC2 instances, setting up IAM permissions following the Principle of Least Privilege, and encrypting sensitive data.

**Q4: If you're using Lambda instead of EC2, does your responsibility change?**
> Yes — with Lambda (PaaS/serverless), AWS also manages OS and runtime patching, so your responsibility shrinks to just your function code and data. The more managed the service, the smaller your security surface.

**Q5: Who is responsible if an EC2 instance gets hacked through an open SSH port?**
> The customer — AWS provides Security Groups as a tool but configuring them is the customer's responsibility. An open port 22 to 0.0.0.0/0 is a customer configuration error, not an AWS failure.

---

## 21. Hands-On Assignment for Today

### Console Navigation Practice:

1. Use the **search bar** to navigate to: EC2, S3, IAM, RDS, Lambda — time yourself. Should be under 5 seconds each.

2. **Change your region** to Singapore → notice how the EC2 dashboard shows different (or zero) instances → switch back to Mumbai.

3. **Pin your 8 most-used services** to the console home page.

4. Open **CloudShell** → run `aws iam list-users` → see your IAM user listed.

### Billing Dashboard Practice:

5. Go to **Billing → Free Tier** → screenshot or note your current usage (should be near 0%).

6. Verify your **Budget alert** from Day 1 is set — if not, create it now.

7. Open **Cost Explorer** → explore the interface even with $0 spend.

### Shared Responsibility:

8. Go to **IAM → Security recommendations** → note what AWS flags as security risks on your fresh account (likely: no MFA on root account).

9. Write a short note covering:
   - "OF the cloud" vs "IN the cloud" — one example each
   - How responsibility changes between EC2 and Lambda
   - The one security mistake from Section 19 and who's responsible

---

## 22. Day 2 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"The AWS Console is organized around a Services menu, a search bar (fastest navigation), and a Region selector — always check your region before creating or looking for resources. The Billing Dashboard lets me monitor costs via Bills, Cost Explorer, Budgets, and Free Tier usage — I set up a $5 budget alert so I'm never surprised. The Shared Responsibility Model means AWS secures the physical infrastructure (OF the cloud) and I secure everything inside it — my data, IAM config, Security Group rules, and OS patches (IN the cloud). The more managed the service (Lambda vs EC2), the less I have to secure myself."*

---

**Next up — Day 3: IAM — Users, Groups, Roles & Policies**
