# Day 18: Load Balancer (ALB)
### *(Distributing Traffic Across Multiple Servers — The Gateway to High Availability)*

> **Roadmap reference:** Week 3, Day 18 — "Load Balancer"

---

## Why This Matters

Right now your Node.js app runs on a single EC2 instance. That means:

```
Current setup:
  All traffic → ONE EC2 instance
        ↓
  That instance goes down (crash, reboot, update)?
  → Your entire application is offline.
  → Every user gets an error.
  → This is called a "Single Point of Failure."
```

A Load Balancer eliminates this single point of failure. It's the component that graduates your app from "something I built" to "something that can survive real production traffic." It's also the gateway concept before Auto Scaling (Day 19), which works hand-in-hand with a Load Balancer.

---

## 1. What is a Load Balancer?

**A Load Balancer is a managed AWS service that sits in front of your application, receives all incoming traffic, and distributes it across multiple backend instances — ensuring no single instance is overwhelmed, and the app stays available even if one instance fails.**

```
WITHOUT Load Balancer:
  User 1 ──────────────────► EC2 Instance (100% of traffic)
  User 2 ──────────────────► EC2 Instance (overloaded)
  User 3 ──────────────────► EC2 Instance (or offline)

WITH Load Balancer:
  User 1 ──► Load Balancer ──► EC2 Instance A  (33% of traffic)
  User 2 ──►                 ──► EC2 Instance B  (33% of traffic)
  User 3 ──►                 ──► EC2 Instance C  (34% of traffic)
```

---

## 2. Types of AWS Load Balancers

AWS offers three types — knowing which to use and why is a common interview question:

```
┌──────────────────────────────────────────────────────────┐
│  ALB (Application Load Balancer)   ← what you learn today │
│  → Works at Layer 7 (HTTP/HTTPS)                           │
│  → Can route based on URL path, headers, host             │
│  → Best for web apps, REST APIs, microservices            │
│  → Most common choice for Node.js backends                 │
├──────────────────────────────────────────────────────────┤
│  NLB (Network Load Balancer)                               │
│  → Works at Layer 4 (TCP/UDP)                              │
│  → Extremely high performance, handles millions of req/sec │
│  → Best for real-time apps, gaming, IoT, raw TCP traffic  │
├──────────────────────────────────────────────────────────┤
│  GLB (Gateway Load Balancer)                               │
│  → For deploying third-party network appliances            │
│  → Firewalls, intrusion detection — advanced use case      │
└──────────────────────────────────────────────────────────┘
```

> For a Node.js/Express REST API (like QueueCare or PulseBloom), **ALB is always the right choice** — it understands HTTP, can route by path, and integrates natively with ECS, EC2, and Lambda.

---

## 3. How ALB Works — The Key Components

```
ALB has three moving parts you configure:

┌──────────────────────────────────────────────────────────┐
│  LISTENER                                                  │
│  → Watches for incoming traffic on a specific port/protocol│
│  → e.g., "Listen on port 80 (HTTP)"                       │
│  → e.g., "Listen on port 443 (HTTPS)"                     │
├──────────────────────────────────────────────────────────┤
│  RULES                                                     │
│  → Decides what to DO with traffic the listener receives  │
│  → e.g., "If path starts with /api → send to this group"  │
│  → e.g., "If path starts with /admin → send to that group"│
│  → Default rule: "forward everything to the default group"│
├──────────────────────────────────────────────────────────┤
│  TARGET GROUP                                              │
│  → The group of backend instances that receive traffic    │
│  → Contains EC2 instances, ECS tasks, or Lambda functions │
│  → Each target gets health checked continuously            │
└──────────────────────────────────────────────────────────┘
```

```
Full request flow:

User's browser
      │
      │  HTTP request: GET /api/users
      ▼
ALB Listener (port 80)
      │
      │  checks rules
      ▼
Rule: "path /api/* → Target Group: nodejs-servers"
      │
      │  picks a healthy instance from the Target Group
      ▼
EC2 Instance B  ← today it gets the request
      │
      │  tomorrow it might be EC2 Instance A or C instead
      ▼
Response sent back through the ALB to the user
```

---

## 4. Health Checks — How ALB Knows Who's Alive

**A Health Check is an automatic periodic request the ALB sends to each instance in a Target Group to verify it's alive and responding correctly.**

```
ALB sends every 30 seconds (configurable):
  GET /health → EC2 Instance A
        ↓
  If response: HTTP 200 → instance is HEALTHY → receives traffic
  If response: HTTP 500, timeout, or connection refused
               → instance is UNHEALTHY → removed from rotation

Example:
  You have 3 instances: A (healthy), B (healthy), C (crashed)
        ↓
  ALB health check detects C is not responding
        ↓
  C is removed from rotation automatically
        ↓
  Traffic splits between A and B only — users never see errors
        ↓
  When C recovers, health check passes again
        ↓
  C is automatically added back to rotation
```

### Add a Health Check Endpoint to Your Express App

```javascript
// Add this to your Express app — simple, fast, reliable
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString()
  });
});
```

> A good health check endpoint should verify not just that Express is running, but that the critical dependencies (database connection, etc.) are also working. For now, a simple `200 OK` is sufficient — in production you'd also check the DB connection.

---

## 5. Creating an ALB — Step by Step

### Step 1: Create a Security Group for the ALB

```
EC2 → Security Groups → Create security group
  Name: alb-sg
  VPC: my-production-vpc

  Inbound Rules:
    HTTP   port 80   from 0.0.0.0/0  (anyone can reach the ALB)
    HTTPS  port 443  from 0.0.0.0/0  (anyone can reach the ALB)

  Outbound Rules: leave default (all traffic allowed)
```

```
Also update your EC2 instances' Security Group:
  Remove: HTTP port 80 from 0.0.0.0/0  (users shouldn't hit EC2 directly)
  Add:    HTTP port 80 from alb-sg      (only the ALB can reach EC2 now)
```

> This is the proper security pattern — **users talk to the ALB, the ALB talks to EC2, users NEVER talk to EC2 directly.** If the ALB's Security Group is `alb-sg`, and EC2's Security Group only allows port 80 from `alb-sg`, then the ALB is the enforced single entry point.

### Step 2: Create a Target Group

```
EC2 → Target Groups → Create target group

  Target type: Instances
  Name: nodejs-target-group
  Protocol: HTTP
  Port: 80
  VPC: my-production-vpc

  Health checks:
    Protocol: HTTP
    Path: /health
    Healthy threshold:   2 (2 consecutive successes = healthy)
    Unhealthy threshold: 2 (2 consecutive failures = unhealthy)
    Timeout:             5 seconds
    Interval:            30 seconds

  → Next → Register Targets
    → Select your EC2 instance(s) → Include as pending below
  → Create target group
```

### Step 3: Create the ALB

```
EC2 → Load Balancers → Create load balancer → Application Load Balancer

  Name: my-production-alb
  Scheme: Internet-facing  ← faces the public internet
  IP address type: IPv4

  Network mapping:
    VPC: my-production-vpc
    Mappings: select BOTH public subnets
      ✓ public-subnet-1a
      ✓ public-subnet-1b
      (ALB needs at least 2 AZs for high availability)

  Security groups:
    → Remove default
    → Add: alb-sg

  Listeners and routing:
    Protocol: HTTP   Port: 80
    Default action: Forward to → nodejs-target-group

  → Create load balancer
```

```
After creation, find the DNS name:
  Load Balancers → your ALB → Description tab
  DNS name: my-production-alb-xxxxxxxxxx.ap-south-1.elb.amazonaws.com
```

---

## 6. Testing Your ALB

```bash
# Test via curl (replace with your ALB DNS name)
curl http://my-production-alb-xxxxxxxxxx.ap-south-1.elb.amazonaws.com/health

# Expected response:
{"status":"healthy","timestamp":"2026-06-22T10:00:00.000Z"}

# Test your API through the ALB
curl http://my-production-alb-xxxxxxxxxx.ap-south-1.elb.amazonaws.com/users
```

```
Check Target Group health:
  EC2 → Target Groups → nodejs-target-group → Targets tab
  → Your instance should show "Healthy" status

If it shows "Unhealthy":
  1. Is your Express app running? (pm2 list)
  2. Does your Express app have a GET /health route returning 200?
  3. Does your EC2 Security Group allow port 80 from alb-sg?
  4. Is the health check path correct (/health)?
```

---

## 7. Path-Based Routing — The ALB's Superpower

One of ALB's most powerful features is routing different URL paths to different backend services — the foundation of microservices architecture:

```
ALB receives all traffic at one DNS name, then splits by path:

  /api/users    → Target Group: users-service     (EC2 cluster A)
  /api/files    → Target Group: files-service      (EC2 cluster B)
  /api/health   → Target Group: health-service     (EC2 cluster C)
```

```
Setting up path-based rules:
  Load Balancers → your ALB → Listeners → View/edit rules
    → Add rule:
        IF: Path is /api/users*
        THEN: Forward to → users-target-group
```

> For your current single-app setup, you don't need this yet — but understanding it exists is why ALB is so popular for Node.js microservices. Each Express service gets its own Target Group, and the ALB routes traffic intelligently based on the URL.

---

## 8. The Architecture You've Now Built

```
                     Internet
                         │
                         │  port 80
                         ▼
              ┌─────────────────────┐
              │  ALB (alb-sg)         │  ← internet-facing, public subnets
              │  my-production-alb    │  ← DNS: xxxx.elb.amazonaws.com
              └─────────────────────┘
                         │
                         │  health-checked, load-balanced
                         ▼
              ┌─────────────────────┐
              │  Target Group         │
              │  nodejs-target-group  │
              └─────────────────────┘
                    │         │
                    ▼         ▼
              ┌──────┐   ┌──────┐
              │ EC2 A │   │ EC2 B │  ← EC2 in public/private subnet
              │ :80   │   │ :80   │  ← only reachable from ALB SG
              └──────┘   └──────┘
                    │         │
                    └────┬────┘
                         ▼
              ┌─────────────────────┐
              │  RDS PostgreSQL       │  ← private subnet, untouched
              └─────────────────────┘
```

---

## 9. ALB vs Your Current Direct EC2 Setup

```
Before ALB (Week 1 setup):
  Users → Elastic IP → EC2 → Nginx → Express
  Single point of failure. One crash = total downtime.

After ALB (today):
  Users → ALB DNS → Target Group → Multiple EC2 instances
  One instance crashes? ALB detects via health check → removes
  it → routes to healthy instances only. Zero user impact.
```

> The Elastic IP you set up on Day 5 is no longer the entry point — the ALB's DNS name is. You'd point your domain (Route 53, Day 29) to the ALB DNS, not to an individual EC2 IP.

---

## 10. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is a Load Balancer and why is it needed?**
> A managed service that distributes incoming traffic across multiple backend instances, eliminating the single point of failure and ensuring the application stays available even if one instance fails.

**Q2: What's the difference between ALB and NLB?**
> ALB operates at Layer 7 (HTTP/HTTPS) and can make routing decisions based on URL paths, headers, and hostnames — ideal for web apps and REST APIs. NLB operates at Layer 4 (TCP/UDP) with extremely high throughput — ideal for real-time, high-performance, or non-HTTP use cases.

**Q3: What is a Target Group?**
> A logical group of backend resources (EC2 instances, ECS tasks, or Lambda functions) that receive traffic from the Load Balancer, with health checks verifying each target's availability before routing traffic to it.

**Q4: What happens when an ALB health check fails for one of the instances?**
> The ALB marks that instance as unhealthy and stops routing traffic to it automatically. Traffic continues flowing to remaining healthy instances. When the failed instance recovers and health checks pass again, it's automatically re-added to rotation.

**Q5: Why should EC2 instances' Security Groups only allow traffic from the ALB's Security Group, not from 0.0.0.0/0?**
> To enforce the ALB as the single, controlled entry point. If EC2 allows traffic from anywhere, users could bypass the ALB entirely (hitting the EC2 IP directly), circumventing load balancing, security rules, and SSL termination.

**Q6: Why does an ALB need to be mapped to at least two Availability Zones?**
> For the ALB itself to be highly available — if one AZ experiences an outage, the ALB continues operating from the other AZ. Mapping to a single AZ would make the Load Balancer itself a single point of failure.

---

## 11. Hands-On Assignment for Today

1. Add a `GET /health` endpoint to your Express app returning `200 OK` — restart via PM2.
2. Create `alb-sg` Security Group (port 80/443 from 0.0.0.0/0).
3. Update your EC2's Security Group — replace port 80 from `0.0.0.0/0` with port 80 from `alb-sg`.
4. Create `nodejs-target-group` with `/health` check path — register your EC2 instance.
5. Create the ALB mapped to both public subnets, forwarding to the target group.
6. Wait for the target to show **"Healthy"** in the Target Group → test via the ALB DNS name.
7. **Simulate a failure:** Stop your Express app (`pm2 stop my-api`) → watch the Target Group status flip to "Unhealthy" in the console → restart it (`pm2 start my-api`) → watch it go back to "Healthy."
8. Write a short note covering:
   - Why the ALB needs 2 subnets across 2 AZs
   - What a Target Group health check does
   - Why EC2 Security Groups should only allow traffic from the ALB SG

---

## 12. Day 18 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"An ALB sits in front of my application in public subnets, receives all traffic, and distributes it across EC2 instances registered in a Target Group. It continuously health-checks each instance — if one fails, traffic automatically routes to the remaining healthy ones. My EC2 instances' Security Groups only allow traffic from the ALB's Security Group, enforcing the ALB as the single entry point. ALB also supports path-based routing, making it ideal for microservices where different URL paths map to different backend services."*

---

**Next up — Day 19: Auto Scaling**
