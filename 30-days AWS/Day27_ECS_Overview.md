# Day 27: ECS Overview — Container Orchestration on AWS
### *(Running Docker Containers in Production Without Managing Servers)*

> **Roadmap reference:** Week 4, Day 27 — "ECS Overview"

---

## Why This Matters

Yesterday you built a Docker image and pushed it to ECR. But a Docker image sitting in ECR does nothing by itself — you need something to **run** it, keep it alive, scale it, and replace it when it crashes. That's exactly what **ECS** does.

ECS is the bridge between "I have a Docker image" and "I have a production-running containerized application on AWS." It's also the deployment model used by most serious Node.js backends at companies — and directly powers how PulseBloom and QueueCare would be deployed at production scale.

---

## 1. What is ECS?

**Amazon ECS (Elastic Container Service) is AWS's fully managed container orchestration service — it runs, manages, scales, and replaces Docker containers automatically.**

```
You provide:         ECS handles:
  Docker image   →     Pulling the image from ECR
  Task config    →     Deciding how much CPU/RAM to allocate
  Service config →     Running N copies, replacing crashed ones
  ALB target     →     Registering containers with Load Balancer
                        Scaling out/in based on demand
                        Health checking each container
                        Zero-downtime deployments (rolling updates)
```

---

## 2. ECS Launch Types — EC2 vs Fargate

ECS can run containers in two modes — this is one of the most important architectural decisions:

```
┌──────────────────────────────────────────────────────────────┐
│  ECS on EC2                                                    │
│  → You manage the EC2 instances (the "cluster")              │
│  → ECS schedules containers onto those EC2 instances          │
│  → You're responsible for: EC2 patching, scaling the cluster, │
│    choosing right instance types, capacity management         │
│  → More control, potentially cheaper at very large scale      │
│  → More operational overhead                                  │
├──────────────────────────────────────────────────────────────┤
│  ECS on Fargate   ← recommended for most teams               │
│  → No EC2 instances to manage — AWS provides the compute      │
│  → You define CPU + memory per container, AWS handles the rest│
│  → No cluster sizing, no EC2 patching, no capacity planning   │
│  → Pay per vCPU/memory used per second                        │
│  → Perfect for: startups, small teams, variable workloads     │
└──────────────────────────────────────────────────────────────┘
```

> **For QueueCare and PulseBloom — and for this course — always use Fargate.** It's the modern, serverless-infrastructure approach to containers. You focus on your app, AWS manages the underlying compute.

---

## 3. Core ECS Concepts — The Four Building Blocks

```
┌─────────────────────────────────────────────────────────────┐
│  CLUSTER                                                      │
│  → The logical grouping of tasks/services                    │
│  → Like a namespace — "my-production-cluster"                │
│  → With Fargate: just a name, no servers to manage           │
├─────────────────────────────────────────────────────────────┤
│  TASK DEFINITION                                              │
│  → Blueprint for running a container                         │
│  → Defines: which image, CPU/RAM, ports, env vars, IAM role  │
│  → Like a docker-compose.yml but for AWS                     │
│  → Versioned: v1, v2, v3 — each deploy creates a new version │
├─────────────────────────────────────────────────────────────┤
│  TASK                                                         │
│  → A single running instance of a Task Definition            │
│  → One or more containers running together                   │
│  → Like a "docker run" — a running container                 │
│  → Short-lived: runs once (for batch jobs) or managed by Service│
├─────────────────────────────────────────────────────────────┤
│  SERVICE                                                      │
│  → Maintains N copies of a Task running at all times         │
│  → Replaces failed tasks automatically                       │
│  → Handles rolling deployments (new image version)           │
│  → Integrates with ALB for traffic distribution              │
└─────────────────────────────────────────────────────────────┘
```

```
Relationship:
  Cluster → contains → Services → run → Tasks (from Task Definitions)

  Task Definition : Task  =  Class : Object (OOP analogy)
  Service : Task  =  PM2 : Node.js process (from Day 7 analogy)
```

---

## 4. Task Definition — The Heart of ECS

```json
{
  "family": "nodejs-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "nodejs-container",
      "image": "ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:1.0",
      "portMappings": [
        { "containerPort": 5000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "NODE_ENV",  "value": "production" },
        { "name": "PORT",      "value": "5000" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-south-1:ACCOUNT:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group":         "/ecs/nodejs-app",
          "awslogs-region":        "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ]
}
```

### Two IAM Roles in ECS — Commonly Confused

```
executionRoleArn  (ECS Task Execution Role)
  → Used by the ECS AGENT (AWS's infrastructure)
  → Needed to: pull image from ECR, write logs to CloudWatch,
               fetch secrets from Secrets Manager
  → AWS provides a managed policy: AmazonECSTaskExecutionRolePolicy

taskRoleArn       (ECS Task Role)
  → Used by YOUR APPLICATION CODE running inside the container
  → Needed to: call S3, DynamoDB, RDS, other AWS services
  → You define this based on what your app actually needs
  → Same pattern as Lambda's Execution Role (Day 22)

Analogy:
  Execution Role = building security badge (to GET INTO the building)
  Task Role      = your work badge (what you can DO once inside)
```

---

## 5. Building the Full ECS Setup — Step by Step

### Step 1: Create the ECS Cluster

```
AWS Console → ECS → Clusters → Create cluster
  Cluster name: production-cluster
  Infrastructure: AWS Fargate (serverless)
  → Create
```

### Step 2: Create the Task Execution Role

```
IAM → Roles → Create role
  Trusted entity: AWS service → Elastic Container Service Task
  Attach policy: AmazonECSTaskExecutionRolePolicy
  Name: ecsTaskExecutionRole
```

### Step 3: Create the Task Role (for your app's AWS access)

```
IAM → Roles → Create role
  Trusted entity: AWS service → Elastic Container Service Task
  Attach policies:
    AmazonS3ReadOnlyAccess     ← if app reads from S3
    AmazonRDSDataServiceAccess ← if app accesses RDS
    SecretsManagerReadWrite    ← if app reads from Secrets Manager
  Name: ecsTaskRole
```

### Step 4: Create the Task Definition

```
ECS → Task Definitions → Create new task definition

  Task definition family: nodejs-app
  Launch type: AWS Fargate
  OS/arch: Linux/X86_64

  Task size:
    CPU:    0.25 vCPU  (256 CPU units)  ← start small
    Memory: 0.5 GB    (512 MB)

  Task roles:
    Task execution role: ecsTaskExecutionRole
    Task role:           ecsTaskRole

  Container details:
    Name:  nodejs-container
    Image: YOUR_ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:1.0
    Port:  5000

    Environment variables:
      NODE_ENV = production
      PORT     = 5000

    Logging:
      Log collection: on
      awslogs-group:  /ecs/nodejs-app
      (CloudWatch log group auto-created)

  → Create
```

### Step 5: Create the Security Group for ECS Tasks

```
EC2 → Security Groups → Create security group
  Name: ecs-tasks-sg
  VPC: my-production-vpc

  Inbound Rules:
    Custom TCP  port 5000  from: alb-sg
    (Only ALB can send traffic to your containers)

  NO inbound from the internet — ALB is the only entry point
```

### Step 6: Create the ECS Service

```
ECS → production-cluster → Services → Create

  Launch type: FARGATE

  Task definition:
    Family:   nodejs-app
    Revision: 1 (latest)

  Service name:     nodejs-service
  Desired tasks:    2  ← run 2 containers at all times

  Networking:
    VPC: my-production-vpc
    Subnets: private-subnet-1a, private-subnet-1b
      ← Fargate tasks in PRIVATE subnets is best practice
         (ALB in public subnet → private Fargate tasks → private RDS)
    Security group: ecs-tasks-sg
    Public IP: TURNED OFF
      ← Fargate in private subnets doesn't need public IP

  Load balancing:
    Load balancer type: Application Load Balancer
    Load balancer:      my-production-alb
    Listener:           HTTP:80
    Target group:
      → Create new target group
      Name: ecs-target-group
      Protocol: HTTP
      Port: 5000
      Health check path: /health

  → Create service
```

```
ECS pulls your image from ECR → starts 2 Fargate tasks →
registers them with ALB → health check passes →
traffic flows through ALB → your containerized app is live
```

---

## 6. Deploying a New Version (Rolling Update)

This is one of ECS's most powerful features — deploy a new image version with zero downtime:

```bash
# Step 1: Build and push new image version
docker build -t my-nodejs-app:2.0 .
docker tag my-nodejs-app:2.0 \
  ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:2.0
docker push \
  ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:2.0
```

```
Step 2: Update the Task Definition
  ECS → Task Definitions → nodejs-app → Create new revision
    Image: ...amazonaws.com/my-nodejs-app:2.0  ← update version
    → Create

Step 3: Update the Service
  ECS → production-cluster → nodejs-service → Update
    Task definition revision: 2 (latest)
    Deployment options:
      Minimum healthy percent: 50  ← always keep ≥50% running
      Maximum percent:         200 ← allow double capacity during deploy
    → Update
```

```
What ECS does during a rolling update:
  1. Starts 2 new v2.0 tasks alongside existing 2 v1.0 tasks
  2. ALB health checks pass on new tasks → adds them to rotation
  3. ECS drains connections from old v1.0 tasks (waits for in-flight requests)
  4. Stops old v1.0 tasks
  5. You now have 2 running v2.0 tasks — zero user impact throughout
```

---

## 7. Auto Scaling for ECS Services

ECS Services support the same Auto Scaling concepts from Day 19, now applied to containers:

```
ECS → production-cluster → nodejs-service → Update
  → Service auto scaling → Edit
    Minimum tasks: 1
    Maximum tasks: 10

    Scaling policy:
      Target tracking → ECSServiceAverageCPUUtilization
      Target value: 60%

Now ECS scales the number of Fargate tasks automatically,
exactly like ASG scaled EC2 instances on Day 19 — but faster
(containers start in seconds, EC2 takes minutes).
```

---

## 8. The Complete Production Architecture

```
                        Internet
                            │
                            │ HTTPS port 443 / HTTP port 80
                            ▼
              ┌─────────────────────────────┐
              │  Application Load Balancer    │  ← public subnets
              │  (alb-sg: 80/443 from 0.0.0.0│    AZ-1a + AZ-1b
              └─────────────────────────────┘
                            │
                    health-checked, load-balanced
                            │
              ┌─────────────▼─────────────┐
              │  ECS Fargate Tasks          │  ← private subnets
              │  nodejs-service (2–10)       │    AZ-1a + AZ-1b
              │  ecs-tasks-sg:5000 from alb-sg│
              │  ┌──────────┐ ┌──────────┐ │
              │  │Container  │ │Container  │ │
              │  │ v2.0      │ │ v2.0      │ │
              │  └──────────┘ └──────────┘ │
              └─────────────────────────────┘
                     │              │
              Secrets Manager    S3/other AWS services
              (DB credentials)   (via Task Role)
                     │
              ┌──────▼──────────────────────┐
              │  RDS PostgreSQL               │  ← private subnets
              │  (Multi-AZ)                   │
              └───────────────────────────────┘
```

> **This is Portfolio Project 4** — Production Deployment with Docker + ECS + Load Balancer. Every piece of this architecture has been built across the entire 30-day roadmap.

---

## 9. ECS vs EC2 vs Lambda — Full Comparison

```
┌────────────────┬────────────────┬────────────────┬────────────────┐
│ Aspect          │ EC2 (Day 4–7)  │ ECS Fargate     │ Lambda (Day 22)│
├────────────────┼────────────────┼────────────────┼────────────────┤
│ Server mgmt     │ You manage     │ AWS manages     │ No servers     │
│ Scaling speed   │ Minutes (AMI)  │ Seconds (image) │ Milliseconds   │
│ Max runtime     │ Unlimited      │ Unlimited       │ 15 minutes     │
│ Packaging       │ AMI / manual   │ Docker image    │ ZIP / inline   │
│ State           │ Persistent     │ Ephemeral       │ Ephemeral      │
│ Idle cost       │ Always billing │ Always billing  │ Zero at zero   │
│ Best for        │ Full control,  │ Containerized   │ Event-driven,  │
│                 │ legacy apps    │ APIs, services  │ short tasks    │
└────────────────┴────────────────┴────────────────┴────────────────┘
```

---

## 10. Common Errors and Fixes

```
Error: Task stops immediately after starting
  Fix 1: Check CloudWatch Logs (/ecs/nodejs-app) — look for app errors
  Fix 2: Container might be crashing on startup (missing env vars,
          wrong port, DB connection failure)
  Fix 3: Health check failing — ensure GET /health returns 200

Error: "CannotPullContainerError: pull access denied for image"
  Fix: ecsTaskExecutionRole is missing ECR pull permissions
       → Attach AmazonECSTaskExecutionRolePolicy to execution role
       → Confirm image URI in task definition is exactly correct

Error: Task running but ALB shows "unhealthy"
  Fix: ecs-tasks-sg doesn't allow inbound port 5000 from alb-sg
       → Add inbound rule: TCP 5000 from alb-sg

Error: "ResourceNotFoundException: Secret not found"
  Fix: Secrets Manager ARN in task definition is wrong
       → Copy the FULL ARN from Secrets Manager exactly

Error: Tasks running in public subnet get no internet access
  Fix: Private subnets need a NAT Gateway (Day 18) for ECR pulls
       → Fargate in private subnet pulls ECR image via NAT or
          VPC Endpoint for ECR
```

---

## 11. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is ECS and what does it orchestrate?**
> ECS is AWS's managed container orchestration service — it runs Docker containers, keeps the desired number of tasks running, replaces failed ones, handles rolling deployments, and integrates with ALB for traffic distribution and Auto Scaling for demand management.

**Q2: What is the difference between a Task Definition and a Service in ECS?**
> A Task Definition is the blueprint defining what containers to run, their image, CPU/RAM, ports, and IAM roles. A Service uses that blueprint to maintain N running tasks at all times, replacing failed ones and handling deployments — like PM2 manages Node.js processes but for containers.

**Q3: What are the two IAM roles in ECS and what is each for?**
> The Task Execution Role is used by AWS's ECS agent to pull images from ECR and write logs to CloudWatch. The Task Role is used by your application code running inside the container to call other AWS services like S3, RDS, or Secrets Manager.

**Q4: Why run ECS Fargate tasks in private subnets?**
> Containers in private subnets aren't directly reachable from the internet — only the ALB (in the public subnet) can route traffic to them. This reduces attack surface, exactly like placing RDS in private subnets on Day 12.

**Q5: How does a rolling update work in ECS?**
> ECS starts new tasks with the updated image alongside existing ones. Once new tasks pass health checks, the ALB shifts traffic to them while draining connections from old tasks. Old tasks are then stopped — resulting in zero downtime during the entire deployment.

**Q6: How is ECS Fargate different from ECS on EC2?**
> Fargate is serverless — no EC2 instances to manage, patch, or scale. You define CPU and memory per task and AWS provides the compute. EC2 mode requires you to manage the underlying cluster EC2 instances yourself, giving more control at the cost of more operational overhead.

---

## 12. Hands-On Assignment for Today

1. **Create an ECS Cluster** (`production-cluster`, Fargate).

2. **Create both IAM roles** — `ecsTaskExecutionRole` and `ecsTaskRole`.

3. **Create a Task Definition** using your Day 26 ECR image — CPU: 256, Memory: 512, port 5000, logging to CloudWatch.

4. **Create `ecs-tasks-sg`** — inbound TCP 5000 from `alb-sg` only.

5. **Create the ECS Service** (`nodejs-service`, desired: 2 tasks) in private subnets, attached to your ALB from Day 18.

6. **Watch the deployment:**
   - ECS → production-cluster → Tasks tab → watch tasks go from PENDING → RUNNING
   - ALB → Target Groups → watch targets go from initial → healthy

7. **Test via ALB DNS** — confirm your containerized app responds.

8. **Simulate a rolling update:**
   - Edit `index.js` (change the response message)
   - Build `my-nodejs-app:2.0`, push to ECR
   - Create new Task Definition revision with `:2.0`
   - Update the Service → watch the rolling deploy

9. Write a short note covering:
   - The difference between a Task and a Service
   - Why Fargate tasks go in private subnets
   - The difference between Task Execution Role and Task Role

---

## 13. Day 27 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"ECS Fargate runs my Docker containers without any EC2 instances to manage. I define a Task Definition — the blueprint with image, CPU, RAM, ports, and IAM roles — and a Service maintains the desired number of running tasks, replacing crashed ones automatically and performing zero-downtime rolling deployments when I push a new image. Fargate tasks run in private subnets, behind an ALB in public subnets, pulling secrets from Secrets Manager via the Task Execution Role. The Task Role gives my application code access to S3, RDS, and other AWS services without hardcoded credentials."*

---

**Next up — Day 28: Elastic Beanstalk**
