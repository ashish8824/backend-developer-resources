# Day 19: Auto Scaling
### *(Making Your App Grow and Shrink Automatically — No Human Required)*

> **Roadmap reference:** Week 3, Day 19 — "Auto Scaling"

---

## Why This Matters

Yesterday's Load Balancer solved the single-point-of-failure problem — but you still manually decide how many EC2 instances exist. What happens when your app gets 10x its usual traffic at midnight? You'd need to wake up, log in, and manually launch more instances. And when traffic drops, those idle instances keep billing you.

**Auto Scaling solves both problems:**
- Traffic spike → automatically adds more instances
- Traffic drops → automatically removes idle instances
- Instance fails → automatically replaces it

This is the concept that makes cloud infrastructure truly "elastic" — the word that's literally in EC2's name.

---

## 1. What is Auto Scaling?

**Auto Scaling is an AWS service that automatically adjusts the number of EC2 instances in your application based on demand — scaling out (adding instances) when load increases, and scaling in (removing instances) when load decreases.**

```
WITHOUT Auto Scaling:
  Normal traffic: 2 instances running (over-provisioned for off-peak)
  Traffic spike:  2 instances → overloaded → users get slow responses
  Manual fix:     You wake up at 2 AM, add more instances

WITH Auto Scaling:
  Normal traffic:  2 instances (right-sized)
  Traffic spike:   2 → 4 → 8 instances automatically (within minutes)
  Traffic drops:   8 → 4 → 2 instances automatically
  Instance fails:  Auto Scaling detects via health check → launches
                   replacement automatically
```

---

## 2. The Three Core Components

```
AUTO SCALING GROUP (ASG)
        │
        ├── LAUNCH TEMPLATE        ← "what kind of instance to launch"
        │     (AMI, instance type,
        │      security group, etc.)
        │
        ├── SCALING POLICIES        ← "when to launch / terminate"
        │     (based on CPU, traffic,
        │      schedule, etc.)
        │
        └── LOAD BALANCER TARGET   ← "register new instances here"
              GROUP INTEGRATION       automatically
```

### Auto Scaling Group (ASG)
**The ASG is the container that manages a fleet of EC2 instances — defining min/max/desired counts and where instances run.**

```
ASG Configuration:
  Minimum capacity:  1  (never go below 1 instance — always on)
  Maximum capacity:  6  (never exceed 6 — cost control)
  Desired capacity:  2  (start with 2, let scaling policies adjust)
```

### Launch Template
**A Launch Template defines exactly what each new auto-scaled instance should look like — the "recipe" for every instance ASG creates.**

```
Launch Template contains:
  AMI ID          → which OS/image (your Day 14 custom AMI)
  Instance type   → t3.micro, t3.medium, etc.
  Key pair        → for SSH access
  Security Groups → which firewall rules to attach
  User Data       → startup script to run on every new instance
                    (install Node.js, pull code, start PM2, etc.)
```

> **This is why you created an AMI on Day 14** — when Auto Scaling launches new instances, it needs an image that already has your app pre-configured. That AMI is the snapshot of your "perfect server" that gets replicated automatically.

---

## 3. Scaling Policies — When to Scale

AWS offers three types of scaling policies — knowing when to use each is a common interview question:

### Type 1: Target Tracking Scaling ← Most Common, Start Here
**Automatically adjusts instance count to keep a specific metric at a target value.**

```
Example:
  Target: Keep average CPU utilization at 50%
        ↓
  CPU drops to 20% → ASG terminates excess instances
  CPU rises to 80% → ASG launches new instances
  CPU stays at 50% → no action

AWS manages all the math — you just say "keep CPU around 50%"
and it figures out how many instances that requires.
```

### Type 2: Step Scaling
**Scale in fixed steps based on how far a metric exceeds thresholds.**

```
Example:
  CPU 60–70%  → add 1 instance
  CPU 70–80%  → add 2 instances
  CPU > 80%   → add 3 instances

More granular control but requires more configuration.
```

### Type 3: Scheduled Scaling
**Scale based on a time schedule, when traffic patterns are predictable.**

```
Example (predictable traffic):
  Every Monday–Friday 9 AM:  scale to 4 instances (work hours)
  Every Monday–Friday 7 PM:  scale down to 2 instances
  Every Saturday/Sunday:      scale to 1 instance (low weekend traffic)
```

---

## 4. User Data — The Auto Scaling Secret Weapon

When Auto Scaling launches a new EC2 instance, it's a fresh, blank Ubuntu machine. You need it to automatically install your app and start serving traffic — **without any human touching it.**

**User Data is a script that runs automatically on every new EC2 instance at first boot.**

```bash
#!/bin/bash
# This script runs automatically on every new instance Auto Scaling launches

# Update packages
apt-get update -y

# Install Node.js (NodeSource LTS)
curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -
apt-get install -y nodejs

# Install PM2 globally
npm install -g pm2

# Pull your latest application code from GitHub
git clone https://github.com/yourusername/your-app.git /home/ubuntu/app

# Navigate to app and install dependencies
cd /home/ubuntu/app
npm install

# Start the app with PM2
pm2 start index.js --name my-api
pm2 startup
pm2 save

# Done — instance is now serving traffic
```

> This User Data script is what makes Auto Scaling truly "automatic" — every new instance is immediately production-ready, with zero manual steps. In real deployments, you'd often use a pre-baked AMI (Day 14) combined with a smaller User Data script that just pulls the latest code and restarts the app.

---

## 5. Auto Scaling + Load Balancer — The Complete Picture

Auto Scaling and ALB work together seamlessly:

```
Step 1: CPU rises above target threshold
        ↓
Step 2: ASG launches a new EC2 instance using the Launch Template
        ↓
Step 3: New instance boots, User Data script runs, app starts
        ↓
Step 4: ASG registers the new instance with ALB Target Group
        ↓
Step 5: ALB runs health check on new instance
        ↓
Step 6: Health check passes → instance starts receiving traffic
        ↓
Step 7: Load is now distributed across all healthy instances

When traffic drops:
Step 1: CPU drops below target threshold
        ↓
Step 2: ASG selects an instance to terminate (usually oldest)
        ↓
Step 3: ALB drains connections from that instance (waits for
         current requests to finish — "connection draining")
        ↓
Step 4: Instance is terminated
        ↓
Step 5: Remaining instances handle the reduced traffic
```

```
         Internet
             │
             ▼
      ┌─────────────┐
      │     ALB       │
      └─────────────┘
             │ distributes traffic
    ┌─────────┼─────────┐
    ▼         ▼         ▼
  EC2-1     EC2-2     EC2-3   ← managed by ASG (min:1 max:6 desired:2-3)
    │         │         │
    └────┬────┘─────────┘
         ▼
    ┌──────────┐
    │ RDS (PG)  │  ← untouched, same as before
    └──────────┘
```

---

## 6. Creating an Auto Scaling Group — Step by Step

### Step 1: Create a Launch Template

```
EC2 → Launch Templates → Create launch template

  Name: nodejs-launch-template
  AMI: [your Day 14 AMI — "node-api-configured-v1"]
       OR Ubuntu Server 22.04 (with User Data to configure it)
  Instance type: t3.micro
  Key pair: my-first-server-key
  Security Groups: your EC2 security group (allows from alb-sg)

  Advanced details → User Data:
    [paste your startup script from Section 4]

  → Create launch template
```

### Step 2: Create the Auto Scaling Group

```
EC2 → Auto Scaling Groups → Create Auto Scaling group

  Name: nodejs-asg
  Launch template: nodejs-launch-template

  → Next: Choose instance launch options
    VPC: my-production-vpc
    Availability Zones and subnets:
      ✓ public-subnet-1a
      ✓ public-subnet-1b
      (spread across 2 AZs for high availability)

  → Next: Configure advanced options
    Load balancing:
      ✓ Attach to an existing load balancer
      → Choose target group: nodejs-target-group
    Health checks:
      ✓ Turn on Elastic Load Balancing health checks
        (ALB health check + EC2 health check combined)

  → Next: Configure group size and scaling
    Desired capacity: 2
    Minimum capacity: 1
    Maximum capacity: 4

    Automatic scaling:
      → Target tracking scaling policy
        Metric: Average CPU utilization
        Target value: 50
        Warm-up time: 300 seconds
        (gives new instances 5 mins to fully start
         before being counted in scaling decisions)

  → Next → Next → Create Auto Scaling group
```

---

## 7. Key Concepts Around Scaling Behavior

### Cooldown Period
```
After Auto Scaling adds instances, it waits (default: 300 seconds)
before evaluating whether to scale again.

Why: New instances take time to start and warm up. Without a
     cooldown, ASG might see CPU still high (because new instances
     haven't started yet) and launch even MORE instances
     unnecessarily, over-scaling and increasing cost.
```

### Scale-In Protection
```
You can protect specific instances from being terminated
during a scale-in event:

EC2 → Auto Scaling Groups → your ASG → Instances tab
  → select an instance → Actions → Set scale-in protection

Use case: Prevent ASG from terminating an instance that's
          currently processing a long background job.
```

### Instance Refresh
```
When you update your Launch Template (e.g., new AMI with updated
app code), existing instances still run the old version.

Instance Refresh replaces ALL running instances gradually
with fresh ones using the new template — zero-downtime deployment.

ASG → Instance refresh → Start instance refresh
  → Minimum healthy percentage: 90%
    (keeps 90% of instances healthy while replacing the rest)
```

---

## 8. Connecting This to Real Projects

```
PulseBloom / QueueCare production deployment:

Normal traffic (weekday afternoons):
  ASG: 2 instances running, CPU ~30%
  Cost: paying for 2 × t3.micro

Evening peak (7–9 PM, users logging mood entries):
  CPU spikes to 70%
  ASG: adds 2 more → 4 instances running
  ALB: traffic distributed across all 4
  Cost: paying for 4 × t3.micro temporarily

Late night (2–5 AM, very low traffic):
  CPU drops to 5%
  ASG: scales down to 1 instance (minimum)
  Cost: paying for just 1 × t3.micro

Result: App always available, cost always proportional to load.
        No waking up at 2 AM to manually scale.
```

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is Auto Scaling, and what problems does it solve?**
> Auto Scaling automatically adjusts the number of EC2 instances based on demand — adding instances during traffic spikes (availability) and removing them during low traffic (cost efficiency), and automatically replacing failed instances (self-healing).

**Q2: What are the three types of scaling policies?**
> Target Tracking (keep a metric like CPU at a set target — simplest, most common), Step Scaling (scale in fixed steps based on threshold ranges — more granular), and Scheduled Scaling (scale based on time — for predictable traffic patterns).

**Q3: What is a Launch Template?**
> A reusable configuration defining what each auto-scaled EC2 instance should look like — AMI, instance type, key pair, security groups, and User Data startup script — so every new instance is identical and automatically configured.

**Q4: What is User Data in the context of Auto Scaling?**
> A script that runs automatically on every new EC2 instance at first boot — used in Auto Scaling to automatically install dependencies, pull the latest app code, and start the application, making each new instance immediately production-ready without manual intervention.

**Q5: What is the cooldown period and why is it important?**
> A wait period after a scaling event during which Auto Scaling doesn't evaluate further scaling. It prevents over-scaling by giving newly launched instances time to start and stabilize before the ASG decides whether more scaling is still needed.

**Q6: How do Auto Scaling and a Load Balancer work together?**
> When ASG launches a new instance, it registers it with the ALB's Target Group automatically. The ALB runs health checks on it and, once healthy, starts routing traffic to it. When ASG terminates an instance, the ALB drains existing connections first, then stops routing to it.

---

## 10. Hands-On Assignment for Today

1. Create a **Launch Template** using your Day 14 AMI (or Ubuntu + User Data script).
2. Create an **Auto Scaling Group** (`nodejs-asg`) across both public subnets, attached to `nodejs-target-group`.
3. Set: Min=1, Desired=2, Max=4 with a **Target Tracking policy** at 50% CPU.
4. Confirm two instances appear in EC2 → Instances — both showing "InService" in the Target Group.
5. **Simulate scaling:** SSH into one instance and run a CPU stress test:
   ```bash
   sudo apt install -y stress
   stress --cpu 4 --timeout 300
   ```
   Watch CloudWatch → EC2 → CPUUtilization rise → ASG launches a 3rd instance.
6. Stop the stress test → watch ASG scale back down after the cooldown.
7. Write a short note covering:
   - Why a Launch Template is needed for Auto Scaling
   - What the cooldown period prevents
   - How ASG and ALB integrate to handle new instance registration

---

## 11. Day 19 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Auto Scaling uses an Auto Scaling Group to maintain a fleet of EC2 instances between configured min and max counts, launching new ones when load increases and terminating them when load drops — based on scaling policies like Target Tracking CPU. A Launch Template defines the recipe for each new instance, including a User Data script that auto-configures the app at boot. Auto Scaling integrates with the ALB automatically — new instances are registered to the Target Group, health-checked, and then traffic flows to them. The result is an app that's both always available and cost-proportional to demand."*

---

## Week 3 Mini Project — Highly Available Node.js Application ✅

Combining Days 15–19, you now have:

```
  ✓ Custom VPC with public/private subnets (Days 15–17)
  ✓ ALB distributing traffic across instances (Day 18)
  ✓ Auto Scaling adjusting capacity dynamically (Day 19)
  ✓ RDS in private subnet (Week 2)
  ✓ S3 for file storage (Week 2)
```

**This is Portfolio Project 4** — a production-grade, highly available Node.js deployment on AWS. Every component in this architecture is something you can walk through confidently in a technical interview.

---

**Next up — Day 20: CloudWatch Metrics**
