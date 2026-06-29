# Day 30: Final Revision, Mock Interview & Resume Update
### *(30 Days of AWS — Everything You've Built, Everything You Can Now Say)*

> **Roadmap reference:** Week 4, Day 30 — "Final Revision, Mock Interview, Resume Update"

---

## The Journey You've Completed

```
Week 1: Foundations (Days 1–7)
  Cloud Computing → Global Infrastructure → IAM → EC2 →
  Security Groups → Node.js on EC2 → PM2 + Nginx deployment

Week 2: Storage & Databases (Days 8–14)
  S3 Basics → S3 Permissions → Node.js + S3 → RDS Overview →
  Create RDS → Node.js + RDS → Backups & Snapshots

Week 3: Networking & Monitoring (Days 15–21)
  VPC → Subnets → Internet Gateway → Route Tables →
  Public vs Private → Load Balancer → Auto Scaling →
  CloudWatch Metrics → CloudWatch Logs

Week 4: Serverless & DevOps (Days 22–29)
  Lambda Introduction → Lambda Functions → API Gateway →
  Serverless Project → Docker → ECS → Elastic Beanstalk →
  Route 53 → CloudFront

Portfolio Projects:
  ✓ Project 1: Node.js REST API (EC2 + PM2 + Nginx)
  ✓ Project 2: File Upload API (S3 + PostgreSQL RDS)
  ✓ Project 3: Serverless URL Shortener (Lambda + API Gateway + DynamoDB)
  ✓ Project 4: Production Deployment (Docker + ECS + ALB)
```

---

## PART 1: RAPID REVISION — Every Concept in One Place

---

## 1. Cloud Computing & AWS Infrastructure (Days 1–2)

```
Cloud Computing:
  → Renting computing resources over the internet
  → Pay-as-you-go, on-demand, scalable, globally available
  → IaaS (EC2), PaaS (Beanstalk), SaaS (Gmail)

AWS Global Infrastructure:
  → Region: geographic cluster of data centers
  → Availability Zone: isolated data center(s) within a Region
  → Edge Location: caching nodes for CloudFront, Route 53

Shared Responsibility Model:
  → AWS secures: hardware, data centers, networking (OF the cloud)
  → You secure: data, IAM, OS patches on EC2, Security Groups (IN the cloud)
```

---

## 2. IAM (Day 3)

```
Users  → individual identities (people or apps)
Groups → collections of users sharing the same policies
Roles  → temporary identities assumed by services/apps (no hardcoded keys)
Policies → JSON documents defining Allow/Deny on Actions + Resources

Principle of Least Privilege:
  → Grant only the minimum permissions needed

Key rules:
  → Never use Root account for daily work
  → EC2/ECS/Lambda access other AWS services via Roles, never keys
  → IAM Policy: attached to User/Group/Role
  → Bucket Policy: attached to the S3 bucket itself
```

---

## 3. EC2 (Days 4–7)

```
EC2 = rented virtual server
AMI  = OS template (blueprint for instances)
Instance Type = hardware size (t3.micro, m5.large)
Key Pair = cryptographic SSH access (never lose .pem file)

Security Group = stateful firewall (allow/deny by port + source)
Elastic IP = static public IP that survives restarts

Day 6/7 Stack:
  EC2 → Nginx (reverse proxy, port 80) → PM2 → Express (port 5000)
```

---

## 4. S3 (Days 8–10)

```
Bucket = globally-unique container for objects
Object = file + metadata + key (flat structure, no real folders)
Storage Classes: Standard → Standard-IA → Glacier → Deep Archive
  (frequent → infrequent → archive → long-term archive)

Permissions layers (most restrictive wins):
  Block Public Access → IAM Policy → Bucket Policy → ACLs

Pre-signed URLs = temporary, time-limited access to private objects

Node.js + S3:
  AWS SDK v3: S3Client + PutObjectCommand
  Multer memoryStorage() → buffer → PutObjectCommand
  Store S3 URL in DB, never store binary files in DB
```

---

## 5. RDS (Days 11–13)

```
RDS = managed relational database (PostgreSQL, MySQL, etc.)
  AWS handles: patching, backups, failover, monitoring
  You handle: schema, queries, connection config, IAM

Multi-AZ = high availability (standby in different AZ, auto-failover)
Read Replica = performance (actively queryable copy for read traffic)
PITR = Point-in-Time Recovery (any second within retention window)

Always in private subnet — only reachable from app tier via Security Group
Connection string: localhost → RDS endpoint (+ sslmode=require)
Connection pool (pg/Prisma): reuse connections, don't open per request
```

---

## 6. VPC & Networking (Days 15–17)

```
VPC = your private network inside AWS (CIDR: 10.0.0.0/16)
Subnet = subdivision of VPC, pinned to one AZ
  Public subnet = Route Table has 0.0.0.0/0 → Internet Gateway
  Private subnet = no IGW route (no direct internet access)

Internet Gateway = door to the internet (one per VPC)
Route Table = traffic map (most specific route wins)

Production layout:
  Public subnets:  ALB, NAT Gateway, Bastion Host
  Private subnets: EC2/ECS tasks, RDS, ElastiCache

Two layers of access control:
  Route Table: "Does a path exist to this subnet?"
  Security Group: "Is this specific traffic allowed to this instance?"
```

---

## 7. Load Balancer & Auto Scaling (Days 18–19)

```
ALB (Layer 7):
  → Listener → Rules → Target Group → EC2/ECS instances
  → Health checks: only routes to healthy targets
  → Path-based routing: /api/* → service A, /admin/* → service B
  → Security: EC2 SG only allows traffic from ALB SG

Auto Scaling Group:
  Min/Desired/Max capacity
  Launch Template: AMI + instance type + SG + User Data
  Scaling policies:
    Target Tracking (simplest): maintain CPU at 50%
    Step Scaling: add N instances when CPU crosses threshold
    Scheduled: scale up Mon-Fri 9AM, down 7PM
  Cooldown period: wait before evaluating next scaling decision
  Integrates with ALB: new instances auto-registered to Target Group
```

---

## 8. CloudWatch (Days 20–21)

```
Metrics = numerical time-series data (CPU%, connections, request count)
  Key EC2 metrics:    CPUUtilization, NetworkIn/Out, StatusCheckFailed
  Key RDS metrics:    DatabaseConnections, FreeStorageSpace, ReadLatency
  Key ALB metrics:    RequestCount, 5XX_Count, TargetResponseTime
  EC2 memory/disk:    Requires CloudWatch Agent (not built-in)

Alarms: OK → ALARM → INSUFFICIENT_DATA
  Actions: SNS notification, Auto Scaling, Lambda trigger

Logs:
  Log Group → Log Streams → Log Events
  CloudWatch Agent ships app logs to CloudWatch
  Structured JSON logging → queryable via Logs Insights
  Metric Filters: log patterns → CloudWatch metrics → alarms

Debugging workflow: Metrics → identify WHEN/WHAT → Logs → find WHY
```

---

## 9. Lambda & API Gateway (Days 22–25)

```
Lambda:
  Serverless: no servers, pay per 100ms execution
  Trigger: API Gateway, S3, SQS, EventBridge, DynamoDB Streams, SNS
  Handler: exports.handler = async (event, context) => { ... }
  Execution Role: IAM role for Lambda to call other AWS services
  Cold start: 200ms-2s first invocation latency
  Limits: 15 min max, 10GB max memory, 50MB ZIP
  Always use: try/catch, structured logging, env vars not hardcoded keys

API Gateway:
  HTTP API (v2): simple, fast, cheap — use for most Node.js backends
  REST API (v1): advanced features (caching, transformation)
  Route: method + path → integration (Lambda)
  Stage: named deployment (prod, staging) with its own URL
  JWT Authorizer: validate Bearer tokens before Lambda runs
  Response: { statusCode, headers, body: JSON.stringify(...) }
  CORS: add Access-Control-Allow-Origin to every response

DynamoDB (URL Shortener project):
  Key-value/document NoSQL — partition key identifies each item
  On-demand pricing: perfect pairing with Lambda (both scale to zero)
  TTL: auto-delete items after expiry timestamp — no cleanup Lambda needed
```

---

## 10. Docker & ECS (Days 26–27)

```
Docker:
  Dockerfile → Image → Container
  Key instructions: FROM, WORKDIR, COPY, RUN, EXPOSE, CMD
  Layer caching: COPY package.json BEFORE COPY . .
  .dockerignore: exclude node_modules, .env, .git
  Multi-stage build: lean production images (~85MB vs ~180MB)
  ECR: AWS private registry for Docker images

ECS Fargate:
  No EC2 servers to manage — AWS provides compute
  Cluster → Task Definition → Service → Tasks
  Task Definition: image, CPU, RAM, ports, env vars, IAM roles
  Two roles:
    Execution Role: ECR pull + CloudWatch logs (AWS agent needs this)
    Task Role: what your APP code can access (S3, RDS, Secrets Manager)
  Service: maintains N running tasks, rolling deployments, health checks
  Fargate tasks in PRIVATE subnets (ALB in public → tasks in private)
  Rolling update: new tasks pass health check → old tasks drained → stopped
```

---

## 11. Elastic Beanstalk, Route 53, CloudFront (Days 28–29)

```
Elastic Beanstalk:
  PaaS: upload code → Beanstalk creates EC2 + ALB + ASG + CloudWatch
  .ebextensions: YAML config for packages, env vars, Nginx config
  Deployment policies: All-at-once → Rolling → Immutable (safest)
  Always use external RDS (never Beanstalk-managed RDS in production)
  eb deploy: one command to deploy; eb logs: view app output

Route 53:
  DNS service: domain → IP/AWS resource
  A Record: domain → IPv4 address
  ALIAS: domain → AWS resource (free, works on root domain)
  CNAME: domain → another domain (not for root, charged per query)
  Routing policies: Simple, Weighted, Latency-based, Failover, Geolocation
  Health checks: auto-failover if primary endpoint goes down

CloudFront:
  CDN: 600+ Edge Locations cache content near users
  Origin: S3, ALB, API Gateway (where content comes FROM)
  Behavior: rules per path pattern (cache settings, origin selection)
  OAC: lets CloudFront access private S3 without making it public
  SSL cert for CloudFront MUST be in us-east-1 (critical rule)
  Cache invalidation: after deploying new assets
```

---

## PART 2: MOCK INTERVIEW — 40 Questions Across All Domains

---

## Foundational (Days 1–4)

```
Q1:  Explain IaaS, PaaS, SaaS with one AWS example each.
Q2:  What is the Shared Responsibility Model? Give one example each side.
Q3:  What is the difference between a Region and an Availability Zone?
Q4:  Why should you never use the AWS Root account for daily work?
Q5:  What is the Principle of Least Privilege? Give a real example.
Q6:  Difference between an IAM Role and an IAM User?
Q7:  What is an AMI? How is it different from a running EC2 instance?
Q8:  What happens if you lose your EC2 .pem key file?
```

## Storage (Days 8–14)

```
Q9:  Does S3 have real folders? Explain.
Q10: Why must S3 bucket names be globally unique?
Q11: What are S3 storage classes? When would you use Glacier?
Q12: What is a pre-signed URL? When would you use it?
Q13: How would you upload a file from Node.js to S3?
     (Mention: Multer + AWS SDK v3 + PutObjectCommand)
Q14: Should you store large binary files in PostgreSQL? Why not?
Q15: What is Multi-AZ in RDS? How is it different from a Read Replica?
Q16: What is Point-in-Time Recovery in RDS?
Q17: Why should RDS always be in a private subnet?
Q18: What changes in your connection string when moving to RDS?
     (localhost → endpoint, sslmode=require)
```

## Networking (Days 15–19)

```
Q19: What is a VPC? Why does it exist?
Q20: What makes a subnet "public" vs "private"?
     (Route Table with 0.0.0.0/0 → IGW, not a setting on the subnet)
Q21: What is an Internet Gateway? What is a Route Table?
Q22: What is a Bastion Host? When would you use one?
Q23: What is the difference between how a Route Table and a Security Group
     control access?
Q24: What is an ALB? What are Listeners, Rules, and Target Groups?
Q25: What is a health check in ALB? What happens when one fails?
Q26: Why should EC2 Security Groups only allow traffic from ALB SG,
     not from 0.0.0.0/0?
Q27: What are the three Auto Scaling policy types?
     (Target Tracking, Step Scaling, Scheduled)
Q28: What is the cooldown period in Auto Scaling? Why is it needed?
```

## Monitoring (Days 20–21)

```
Q29: Why doesn't EC2 send memory metrics to CloudWatch by default?
     (Hypervisor can't see inside guest OS RAM — needs CloudWatch Agent)
Q30: What is a Metric Filter in CloudWatch Logs?
Q31: Describe your debugging workflow when 5xx errors spike.
     (Metrics → identify time → Logs Insights → find error → cross-reference)
Q32: What are the three CloudWatch Alarm states?
```

## Serverless & Containers (Days 22–27)

```
Q33: What is a Lambda cold start? How do you mitigate it?
Q34: Why must a Lambda return body as a string, not an object?
Q35: What is the difference between HTTP API and REST API in API Gateway?
Q36: Why use DynamoDB instead of RDS for a URL Shortener?
Q37: What is DynamoDB TTL?
Q38: What is the difference between a Docker Image and a Container?
Q39: Why copy package.json before source code in a Dockerfile?
     (Layer caching: npm install only reruns when dependencies change)
Q40: What are the two IAM roles in ECS? What is each for?
     (Execution Role: ECR pull + logs; Task Role: app's AWS access)
```

---

## PART 3: THE SYSTEM DESIGN ANSWER

**"Walk me through how you'd deploy a Node.js backend on AWS."**

This is the most common AWS architecture question for backend developers. Here is the complete answer using everything you've learned:

```
"I'd design it in layers:

NETWORKING:
  Custom VPC with public and private subnets across two AZs
  for high availability. Public subnets hold the Load Balancer,
  private subnets hold the application and database.

COMPUTE:
  Containerize the Node.js app with Docker, push to ECR.
  Run ECS Fargate tasks in private subnets — no EC2 to manage.
  ALB in public subnets distributes traffic, health-checks
  each container, and is the only entry point.
  ECS Auto Scaling adds/removes tasks based on CPU.

DATABASE:
  RDS PostgreSQL in private subnets, Multi-AZ for high
  availability. Only ECS tasks' Security Group can reach it
  on port 5432. Automated 7-day backups + PITR enabled.
  Database credentials in Secrets Manager, injected into
  ECS tasks via the Task Definition — never hardcoded.

STORAGE:
  S3 for file uploads — Node.js uploads via AWS SDK,
  stores only the S3 URL in PostgreSQL.
  Private bucket, accessed via pre-signed URLs for user downloads.

SECURITY:
  IAM roles everywhere: ECS Execution Role for ECR pulls,
  ECS Task Role for S3/Secrets Manager access.
  Security Groups layered: internet → ALB SG → ECS SG → RDS SG.

MONITORING:
  CloudWatch Logs for all container output, structured JSON.
  Alarms on 5xx error rate, CPU, RDS free storage.
  Dashboard showing all critical metrics.

DNS & CDN:
  Route 53 points api.myapp.com to the ALB via ALIAS record.
  CloudFront serves static assets from S3 globally.
  ACM SSL certificate for HTTPS.

DEPLOYMENT:
  GitHub Actions builds the Docker image, pushes to ECR,
  updates the ECS Task Definition, triggers a rolling deploy.
  Zero downtime — new tasks pass health checks before old
  tasks are drained."
```

---

## PART 4: RESUME UPDATE

---

## AWS Skills to Add to Your Resume

```
Cloud & Infrastructure:
  AWS (EC2, S3, RDS, VPC, Route 53, CloudFront)
  Docker, Amazon ECS (Fargate), Amazon ECR
  Elastic Load Balancing (ALB), Auto Scaling
  AWS Lambda, Amazon API Gateway, Amazon DynamoDB

DevOps & Monitoring:
  AWS CloudWatch (Metrics, Logs, Alarms)
  CI/CD with GitHub Actions → ECR → ECS
  Infrastructure as Code awareness (Terraform/CDK — learn next)

Security:
  IAM (Users, Roles, Policies, Least Privilege)
  AWS Secrets Manager
  VPC Security Groups, Network ACLs
  S3 Bucket Policies, OAC for CloudFront
```

---

## Portfolio Projects — How to Describe Them

### Project 1: Node.js REST API Deployment
```
"Deployed a production-grade REST API on AWS EC2 with a
reverse proxy (Nginx) and process management (PM2) for
zero-downtime operation. Configured Security Groups,
Elastic IP, and SSH key pair authentication."
```

### Project 2: File Upload API (S3 + RDS)
```
"Built a file upload service using Node.js + AWS SDK v3,
storing uploads in S3 and metadata in PostgreSQL RDS.
Implemented private S3 access, connection pooling, and
SSL-secured database connections. Deployed on EC2 behind Nginx."
```

### Project 3: Serverless URL Shortener
```
"Designed and deployed a fully serverless URL shortener
using AWS Lambda (Node.js), API Gateway (HTTP API), and
DynamoDB. Implemented automatic link expiration via
DynamoDB TTL, click tracking with fire-and-forget async
updates, and URL validation. Zero infrastructure to manage,
near-zero cost at rest."
```

### Project 4: Production Deployment (Docker + ECS)
```
"Containerized a Node.js backend using Docker with multi-stage
builds and pushed to Amazon ECR. Deployed on ECS Fargate with
ALB for load balancing, Auto Scaling for demand management,
and CloudWatch for monitoring. Architecture uses VPC with
public/private subnet isolation, IAM Task Roles for AWS service
access, and Secrets Manager for credential management."
```

---

## Talking Points for Interviews

```
On security:
  "I follow the Principle of Least Privilege — EC2/ECS tasks
   access S3 and RDS via IAM Roles, never hardcoded credentials.
   Security Groups are layered so only the ALB can reach app servers,
   and only app servers can reach the database."

On cost awareness:
  "I set up CloudWatch billing alarms and budget alerts to catch
   unexpected costs early. I choose the right service for the
   workload — Lambda for event-driven tasks to minimize idle cost,
   RDS Stopped state during non-work hours in dev environments."

On availability:
  "I deploy across multiple AZs — RDS in Multi-AZ mode, ECS tasks
   in subnets across AZ-1a and AZ-1b, ALB spanning both. If one AZ
   fails, the ALB automatically routes to the other."

On monitoring:
  "I use structured JSON logging so every log event is queryable
   in CloudWatch Logs Insights. Metric Filters convert error logs
   into CloudWatch metrics I can alarm on, reducing MTTR when
   something goes wrong in production."
```

---

## PART 5: WHAT TO LEARN NEXT

```
Immediate next steps (build on this foundation):

1. AWS Certified Cloud Practitioner (CLF-C02)
   → You already know 80% of the content
   → Practice at: https://aws.amazon.com/certification/

2. AWS Certified Solutions Architect – Associate (SAA-C03)
   → Deep dive on VPC, IAM, S3, EC2, RDS architecture decisions
   → The most valuable AWS cert for backend developers

3. Infrastructure as Code
   → Terraform: provision all your AWS resources as code
   → AWS CDK: define AWS infrastructure in TypeScript/Python
   → Stop clicking in the console, start versioning infrastructure

4. CI/CD Pipeline
   → GitHub Actions → build Docker image → push to ECR → deploy to ECS
   → Full automated deployment pipeline for your portfolio projects

5. Kubernetes (EKS)
   → AWS's managed Kubernetes service
   → The next level beyond ECS for container orchestration

6. Advanced Serverless
   → AWS Step Functions (orchestrate Lambda workflows)
   → EventBridge (event-driven architectures at scale)
   → SNS + SQS patterns (decoupled microservices)
```

---

## PART 6: THE 30-DAY SUMMARY MAP

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOUR AWS KNOWLEDGE MAP                         │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│  FOUNDATION   │   STORAGE     │  NETWORKING   │   SERVERLESS/OPS  │
│  Days 1-7     │   Days 8-14   │  Days 15-21   │   Days 22-29      │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ Cloud Basics  │ S3 + Policies │ VPC + Subnets │ Lambda + API GW   │
│ IAM           │ Node.js + S3  │ IGW + Routes  │ DynamoDB (TTL)    │
│ EC2 + SSH     │ RDS Setup     │ ALB           │ Docker + ECR      │
│ Security Grp  │ Node.js + RDS │ Auto Scaling  │ ECS Fargate       │
│ Elastic IP    │ Backups       │ CloudWatch    │ Elastic Beanstalk │
│ Node.js + EC2 │               │ Logs Insights │ Route 53          │
│ PM2 + Nginx   │               │               │ CloudFront        │
└──────────────┴──────────────┴──────────────┴────────────────────┘

Portfolio Projects:
  P1: EC2 + PM2 + Nginx        ← manual deployment
  P2: S3 + RDS + Node.js       ← storage & database
  P3: Lambda + API GW + DynamoDB ← fully serverless
  P4: Docker + ECS + ALB        ← container production
```

---

## Day 30 Assignment — Your Final Tasks

```
1. SELF-TEST (1 hour)
   Pick 10 random questions from the mock interview section.
   Answer each out loud, without notes, in 2-3 minutes.
   If you stumble — re-read that day's file.

2. ARCHITECTURE DRAW (30 mins)
   On paper or a whiteboard, draw the complete production
   architecture from memory:
   Route 53 → CloudFront → ALB → ECS → RDS + S3
   Label every Security Group, subnet, and IAM role.
   If you can draw it, you understand it.

3. RESUME UPDATE (30 mins)
   Add all four portfolio projects with the descriptions above.
   Add the AWS skills list to your technical skills section.
   Update: ashish-anand.vercel.app and github.com/ashish8824

4. LINKEDIN UPDATE (15 mins)
   Add "AWS" to your skills section.
   Update your "About" section to mention cloud deployment.
   Post a short update: "Completed a 30-day AWS deep dive —
   deployed Node.js APIs on EC2, ECS Fargate, and Lambda.
   Happy to discuss architecture or answer questions."

5. GITHUB (30 mins)
   Push all four portfolio project codebases.
   Add READMEs with architecture diagrams and setup instructions.
   Include screenshots of the AWS console setup.
```

---

## Final Goal — Can You Own This?

By the end of Day 30, you should be able to walk into any backend developer interview and say:

> *"I've deployed production-grade applications on AWS across four different paradigms: traditional server deployment on EC2 with Nginx and PM2, containerized microservices on ECS Fargate behind an ALB with Auto Scaling, fully serverless APIs using Lambda and API Gateway, and managed deployments via Elastic Beanstalk. I understand VPC networking, IAM security, CloudWatch monitoring, S3 storage, and RDS database management — and I can design, deploy, debug, and cost-optimize real AWS architectures."*

---

## 🎉 30 Days Complete — You Did It.

```
Day 1:  "What is cloud computing?"
Day 30: Production-grade AWS architect.

What changed:
  ✓ 4 Portfolio Projects deployed
  ✓ 30 AWS concepts mastered
  ✓ 40 interview questions you can answer
  ✓ Resume updated with real projects
  ✓ Architecture you can draw from memory

What's next:
  → Apply to backend/cloud roles with confidence
  → AWS Cloud Practitioner certification (you're ready)
  → Terraform for Infrastructure as Code
  → Keep building — AWS knowledge compounds fast
```

---

**You started with "What is Cloud Computing?" and ended with
a full production architecture. That's the 30-day journey.
Now go build something real with it.**
