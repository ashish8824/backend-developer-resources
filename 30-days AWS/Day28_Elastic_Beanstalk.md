# Day 28: Elastic Beanstalk
### *(Deploy Your App in Minutes — AWS Manages the Infrastructure Automatically)*

> **Roadmap reference:** Week 4, Day 28 — "Elastic Beanstalk"

---

## Why This Matters

You've now built infrastructure manually — VPC, EC2, ALB, Auto Scaling, ECS — and you deeply understand how each piece works. **Elastic Beanstalk takes everything you've learned and automates it into a single deployment command.**

It's important to learn it AFTER understanding the underlying pieces (which is why it's Day 28 and not Day 1) — because Beanstalk creates all those same components behind the scenes, and when something goes wrong, you need to know what you're looking at.

---

## 1. What is Elastic Beanstalk?

**AWS Elastic Beanstalk is a PaaS (Platform as a Service) that automatically handles the deployment, scaling, load balancing, monitoring, and health management of your application — you just upload your code.**

```
WITHOUT Beanstalk (what you've done Days 4–27):
  You manually create:
    VPC → Subnets → Security Groups → EC2 → ALB →
    Target Groups → Auto Scaling → CloudWatch Alarms →
    IAM Roles → RDS → Nginx → PM2 → deploy code

WITH Beanstalk:
  You run:
    eb deploy
  Beanstalk creates all of the above automatically.
  You focus entirely on writing code.
```

> **The key insight:** Beanstalk isn't magic — it's the same EC2, ALB, Auto Scaling, and CloudWatch you've already learned. It just creates and manages them for you. This is why learning the fundamentals first makes you a far better Beanstalk user — you can inspect, debug, and override what it does.

---

## 2. Beanstalk vs The Alternatives — When to Use What

```
┌───────────────────┬─────────────────────────────────────────────┐
│ Service            │ Best For                                     │
├───────────────────┼─────────────────────────────────────────────┤
│ Elastic Beanstalk  │ Quick deployments, small teams, startups,   │
│                    │ developers who want minimal DevOps overhead. │
│                    │ "I want to deploy fast without thinking      │
│                    │  about infrastructure."                      │
├───────────────────┼─────────────────────────────────────────────┤
│ ECS Fargate        │ Containerized apps, teams who use Docker,   │
│ (Day 27)           │ need fine-grained container control,        │
│                    │ multi-service microservice architectures.    │
├───────────────────┼─────────────────────────────────────────────┤
│ EC2 + Manual       │ Full custom control, legacy apps, specific  │
│ (Days 4–7)         │ OS/software requirements.                   │
├───────────────────┼─────────────────────────────────────────────┤
│ Lambda + API GW    │ Event-driven, short tasks, serverless.      │
│ (Days 22–25)       │ Cost near zero at zero traffic.             │
└───────────────────┴─────────────────────────────────────────────┘
```

---

## 3. Core Beanstalk Concepts

```
APPLICATION
  → The top-level container — like a project name
  → e.g., "pulsebloom-backend"
  → Can hold multiple environments

ENVIRONMENT
  → A running deployment of your application
  → e.g., "pulsebloom-backend-prod" and "pulsebloom-backend-staging"
  → Each environment has its own URL, EC2 instances, ALB, etc.
  → Two types:
      Web server environment  → handles HTTP traffic (what you'll use)
      Worker environment      → processes SQS queues (background jobs)

PLATFORM
  → The runtime Beanstalk uses
  → e.g., "Node.js 20 running on 64bit Amazon Linux 2023"
  → Beanstalk maintains these platforms, applies OS/runtime patches

APPLICATION VERSION
  → A specific deployment package (ZIP of your code)
  → Stored in S3 automatically
  → Versioned — you can roll back to any previous version instantly
```

```
APPLICATION: pulsebloom-backend
    │
    ├── ENVIRONMENT: pulsebloom-prod
    │     → EC2 instances behind ALB
    │     → URL: pulsebloom-prod.ap-south-1.elasticbeanstalk.com
    │     → Running version: v2.3
    │
    └── ENVIRONMENT: pulsebloom-staging
          → Separate EC2 instances
          → URL: pulsebloom-staging.ap-south-1.elasticbeanstalk.com
          → Running version: v2.4 (being tested before prod)
```

---

## 4. Setting Up Elastic Beanstalk CLI

The **EB CLI** is the most efficient way to work with Beanstalk:

```bash
# Install EB CLI
pip install awsebcli --break-system-packages

# Verify
eb --version
# EB CLI 3.x.x
```

---

## 5. Deploying a Node.js App to Beanstalk

### Your Application Structure

```
my-app/
  ├── index.js          ← your Express app
  ├── package.json
  ├── package-lock.json
  └── .ebextensions/    ← optional Beanstalk config directory
        └── nodecommand.config
```

### The Express App (Beanstalk-Ready)

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const app = express();

app.use(express.json());

// Beanstalk uses PORT environment variable — always use process.env.PORT
const PORT = process.env.PORT || 8080;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Elastic Beanstalk!',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION || '1.0'
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.get('/users', (req, res) => {
  res.json({ users: [{ id: 1, name: 'Ashish' }] });
});

// Must listen on process.env.PORT — Beanstalk sets this automatically
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

> **Critical Beanstalk rule:** Always use `process.env.PORT` as your port — Beanstalk injects this automatically. Hardcoding port 3000 or 5000 will cause your app to fail the health check because Beanstalk's Nginx proxy sends traffic to the port it defines.

### package.json — The start script matters

```json
{
  "name": "my-beanstalk-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "dependencies": {
    "express": "^4.18.0",
    "dotenv": "^16.0.0"
  }
}
```

> Beanstalk runs `npm start` automatically — your `start` script must exist and run your server. If it's missing, Beanstalk will fail to start.

### Initialize and Deploy

```bash
cd my-app

# Initialize Beanstalk for this project
eb init

# Interactive prompts:
#   Region: ap-south-1
#   Application name: my-nodejs-app
#   Platform: Node.js
#   Platform version: Node.js 20 running on 64bit Amazon Linux 2023
#   CodeCommit: No
#   SSH: Yes → select your key pair

# Create your first environment and deploy
eb create production-env \
  --instance-type t3.micro \
  --elb-type application \
  --region ap-south-1

# This takes 3–5 minutes — Beanstalk creates:
#   VPC (or uses default), Security Groups, EC2 instance,
#   ALB, Target Group, Auto Scaling Group, CloudWatch Alarms
#   ... all automatically

# Open your app in the browser
eb open

# Deploy code changes in future
eb deploy
```

---

## 6. Configuration — .ebextensions

**`.ebextensions`** is a folder where you place YAML config files to customize Beanstalk beyond the defaults — install packages, set environment variables, run commands, configure Nginx:

```yaml
# .ebextensions/nodecommand.config
option_settings:
  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "npm start"
    NodeVersion: 20

  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    PORT: 8080
```

```yaml
# .ebextensions/packages.config
packages:
  yum:
    git: []
    postgresql15: []    # install psql client for debugging

commands:
  01_npm_install:
    command: "npm install --production"
    cwd: "/var/app/current"
```

```yaml
# .ebextensions/https.config
# Redirect HTTP to HTTPS
option_settings:
  aws:elb:listener:443:
    ListenerProtocol: HTTPS
    SSLCertificateId: arn:aws:acm:ap-south-1:ACCOUNT:certificate/xxxx
    InstanceProtocol: HTTP
    InstancePort: 8080
```

---

## 7. Environment Variables — The Beanstalk Way

Never hardcode secrets. Set them in Beanstalk's environment:

```bash
# Set via CLI
eb setenv \
  DB_HOST=your-rds-endpoint.rds.amazonaws.com \
  DB_NAME=appdb \
  DB_USER=postgres \
  DB_PASSWORD=your-password \
  NODE_ENV=production

# Or via console:
# Beanstalk → your environment → Configuration → Software → Environment properties
```

```
These are available in your app as process.env.DB_HOST, etc.
They're encrypted at rest and NOT visible in your code or repo.
```

> **For database passwords specifically:** Use Secrets Manager (like you learned with ECS on Day 27) instead of plain environment variables — Beanstalk can reference Secrets Manager values via `.ebextensions`. This is the production-secure approach.

---

## 8. Deployment Policies — How Beanstalk Updates Your App

```
ALL AT ONCE (default, fastest, has downtime):
  → Deploy new version to ALL instances simultaneously
  → App is offline during deployment (few seconds)
  → ✓ Fast  ✗ Brief downtime
  → Good for: dev/staging environments

ROLLING:
  → Deploy to a batch of instances at a time
  → App stays mostly up — only the batch being updated is down
  → ✓ No full downtime  ✗ Reduced capacity during deploy
  → Good for: apps that can handle fewer instances briefly

ROLLING WITH ADDITIONAL BATCH:
  → Launches extra instances first, deploys to them,
    then removes old ones — capacity never drops
  → ✓ Zero capacity reduction  ✗ Slightly more expensive
  → Good for: production apps that can't lose capacity

IMMUTABLE (safest, slowest):
  → Launches a completely new set of instances with new version
  → Switches traffic all at once when all new instances healthy
  → If anything fails: revert by switching back instantly
  → ✓ Safest, zero downtime  ✗ Takes longest, costs most
  → Good for: critical production deployments
```

```bash
# Set deployment policy
eb config
# → Find DeploymentPolicy under aws:elasticbeanstalk:command
# → Change to: Rolling, RollingWithAdditionalBatch, or Immutable
```

---

## 9. Monitoring and Logs

```bash
# View recent logs from all instances
eb logs

# Stream live logs to your terminal
eb logs --all --stream

# SSH into the running instance
eb ssh

# Check environment health
eb health

# View CloudWatch metrics in browser
eb console
```

```
Beanstalk automatically creates CloudWatch alarms for:
  → Environment health (Green/Yellow/Red)
  → EC2 CPUUtilization
  → Application latency
  → HTTP 5xx error rate

You can view all of these in:
  Beanstalk console → your environment → Monitoring tab
```

### Rolling Back a Bad Deployment

```bash
# List all deployed versions
eb appversion

# Deploy a specific previous version
eb deploy --version "v1.2-20260620"

# Or via console:
# Beanstalk → Application Versions → select old version → Deploy
```

> **This is one of Beanstalk's most underrated features** — instant rollback to any previous version in seconds. With manual EC2 deployments, rolling back means re-deploying old code manually.

---

## 10. Connecting to RDS from Beanstalk

```
Option A: Beanstalk-managed RDS (not recommended for production)
  → Beanstalk creates and manages an RDS instance
  → RDS is tied to the Beanstalk environment
  → If you delete the environment, RDS is ALSO deleted
  → Never use this for production data

Option B: External RDS (correct approach — what you built on Day 12)
  → Create RDS independently (as you did on Day 12)
  → Set DB connection env vars in Beanstalk
  → Beanstalk EC2 instances connect to your standalone RDS
  → RDS survives environment deletion, updates, redeployments
```

```bash
# Set the external RDS connection details
eb setenv \
  DB_HOST=your-rds-endpoint.ap-south-1.rds.amazonaws.com \
  DB_PORT=5432 \
  DB_NAME=appdb \
  DB_USER=postgres \
  DB_PASSWORD=your-secure-password
```

---

## 11. What Beanstalk Creates Behind the Scenes

After `eb create`, inspect what AWS actually built:

```
EC2 → Instances:
  → t3.micro running your Node.js app (with Nginx proxy in front)

EC2 → Load Balancers:
  → ALB created by Beanstalk (same as Day 18)

EC2 → Auto Scaling Groups:
  → ASG with min/max/desired config (same as Day 19)

CloudWatch → Alarms:
  → CPU, health, latency alarms (same as Day 20)

S3 → Buckets:
  → "elasticbeanstalk-ap-south-1-ACCOUNT" bucket storing your ZIPs

IAM → Roles:
  → "aws-elasticbeanstalk-ec2-role" (auto-created execution role)
  → "aws-elasticbeanstalk-service-role" (Beanstalk's own role)
```

> **This is exactly why learning the components first matters.** When a Beanstalk health check turns red, you know to look at the ALB Target Group. When CPU is high, you know to check Auto Scaling. The Beanstalk abstraction hides the wiring but doesn't change what's underneath.

---

## 12. Beanstalk vs ECS — Which Should You Use?

```
USE BEANSTALK when:
  ✓ Solo developer or small team
  ✓ You want to deploy quickly with minimal DevOps knowledge
  ✓ Your app is a simple web server (not microservices)
  ✓ You don't need Docker/container-specific features
  ✓ You want managed platform updates (Node.js version, OS patches)

USE ECS FARGATE when:
  ✓ You use Docker and want container-level control
  ✓ Building microservices (multiple services, multiple images)
  ✓ Need to specify exact CPU/memory per container
  ✓ Want a consistent local-to-production environment via Docker
  ✓ Team has DevOps maturity and wants fine-grained control

REAL WORLD:
  Startups early on  → Beanstalk (ship fast, worry about infra later)
  Growing product    → Migrate to ECS as complexity grows
  Both are valid     → Choose based on team size and ops maturity
```

---

## 13. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is Elastic Beanstalk and how does it work?**
> Beanstalk is AWS's PaaS — you upload your application code (a ZIP), and Beanstalk automatically provisions and manages all the underlying infrastructure: EC2, ALB, Auto Scaling, CloudWatch alarms, and security groups. You focus on code, Beanstalk manages the infrastructure.

**Q2: What does Beanstalk create under the hood?**
> The same components you'd create manually: EC2 instances running your app behind Nginx, an ALB distributing traffic, an Auto Scaling Group for elasticity, and CloudWatch alarms for monitoring — all automatically configured and linked.

**Q3: What is the difference between a Beanstalk Application and an Environment?**
> An Application is the top-level project container. An Environment is a specific running deployment of that application — you'd typically have separate environments for production and staging, each with their own infrastructure and URL.

**Q4: Why should you use an external RDS instead of Beanstalk-managed RDS?**
> Beanstalk-managed RDS is tied to the environment — deleting or recreating the environment deletes the database. External RDS is independent, survives environment changes, and is the correct approach for any production data.

**Q5: What are Beanstalk deployment policies and which is safest?**
> Deployment policies control how new code reaches running instances. All-at-once is fastest but has downtime. Rolling updates instances in batches. Rolling with additional batch maintains full capacity. Immutable (safest) launches a fresh set of instances with the new version before switching traffic, allowing instant rollback.

**Q6: When would you choose Beanstalk over ECS Fargate?**
> Beanstalk is better for simpler apps, small teams, or when you want to deploy quickly without thinking about Docker or infrastructure design. ECS is better when you're already using Docker, building microservices, or need fine-grained control over container resources and networking.

---

## 14. Hands-On Assignment for Today

1. **Install the EB CLI** (`pip install awsebcli --break-system-packages`).

2. **Create a Beanstalk-ready Express app** with `process.env.PORT` and a `/health` endpoint.

3. **Initialize and create an environment:**
   ```bash
   eb init    # configure project
   eb create production-env --instance-type t3.micro
   ```

4. **Open the URL** (`eb open`) — confirm your app loads.

5. **Set environment variables:**
   ```bash
   eb setenv NODE_ENV=production APP_VERSION=1.0
   ```
   Confirm they're accessible via `process.env` in the app.

6. **Make a code change** (update the response message) → `eb deploy` → refresh the browser.

7. **Check Beanstalk's auto-created infrastructure:**
   - EC2 → find the Beanstalk instance
   - ALB → find the Beanstalk load balancer
   - Auto Scaling → find the Beanstalk ASG

8. **Terminate when done** (`eb terminate production-env`) to stop billing.

9. Write a short note covering:
   - What Beanstalk creates automatically vs what you'd create manually on Day 4–19
   - Why you'd use external RDS instead of Beanstalk-managed RDS
   - One scenario where you'd choose Beanstalk over ECS

---

## 15. Day 28 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Elastic Beanstalk is a PaaS that provisions and manages all AWS infrastructure — EC2, ALB, Auto Scaling, CloudWatch — automatically when I upload my code. I configure it via .ebextensions YAML files, set environment variables for config and secrets, and choose a deployment policy (All-at-Once for dev, Immutable for production) to control how updates roll out. Beanstalk doesn't create new infrastructure — it creates the same EC2, ALB, and ASG I'd build manually, which is why understanding those services first makes me a much more effective Beanstalk user."*

---

**Next up — Day 29: Route 53 & CloudFront**
