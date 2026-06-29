# Day 26: Docker Basics
### *(Package Your App Once, Run It Anywhere — The Foundation of Modern Deployment)*

> **Roadmap reference:** Week 4, Day 26 — "Docker Basics"

---

## Why This Matters

Every deployment approach you've learned so far has a hidden fragility:

```
Day 6/7:  "Install Node.js on EC2, run Express"
  Problem: Works on your EC2. What if you need 10 instances?
           Each one needs Node.js installed, npm packages
           downloaded, config set up manually. Inconsistent.
           "Works on my machine" is still possible.

Day 19:   "Auto Scaling launches EC2 from an AMI"
  Problem: AMI is a 20GB disk image. Slow to create, slow to
           push. If dependencies change, you bake a new AMI.

Docker fixes both: your app + ALL its dependencies are packaged
into a single, portable, 100–200MB image that runs identically
everywhere — your laptop, EC2, ECS, anywhere.
```

Docker is also the foundation of Day 27 (ECS) and Portfolio Project 4 — so getting this right today is critical.

---

## 1. What is Docker?

**Docker is a platform for packaging applications and their dependencies into lightweight, portable containers that run consistently across any environment.**

```
WITHOUT Docker:
  Developer's laptop:  Node 18, npm 9, Ubuntu 22
  EC2 instance:        Node 16, npm 8, Ubuntu 20
  Result: "It works on my machine but not in production"

WITH Docker:
  Developer builds a container image:
    └── Node 20 + your code + all npm packages + config
  That exact same image runs:
    → On your laptop
    → On EC2
    → On ECS Fargate (Day 27)
    → On any cloud, anywhere
  Result: "Works everywhere, always, identically"
```

### VM vs Container — The Key Distinction

```
VIRTUAL MACHINE (EC2):
  ┌──────────────────────────┐
  │  Your App                  │
  │  Node.js Runtime           │
  │  OS (Ubuntu 22 — full)     │  ← 20 GB, boots in ~60 seconds
  │  Virtual Hardware           │
  └──────────────────────────┘
  Heavy: full OS per app

CONTAINER (Docker):
  ┌──────────────────────────┐
  │  Your App                  │
  │  Node.js Runtime           │  ← shares the HOST OS kernel
  └──────────────────────────┘  ← 100-200 MB, starts in ~1 second
  Lightweight: shares host OS, isolated process
```

> **Analogy:** A VM is like renting a whole house. A container is like renting a single furnished room in a shared building — you have your own private space, but you share the building's plumbing, electricity, and roof (the OS kernel).

---

## 2. Core Docker Concepts

```
┌──────────────────────────────────────────────────────────┐
│  DOCKERFILE                                               │
│  → A text file with instructions for BUILDING an image   │
│  → "Recipe": FROM, RUN, COPY, EXPOSE, CMD                │
├──────────────────────────────────────────────────────────┤
│  IMAGE                                                    │
│  → A read-only snapshot built from a Dockerfile          │
│  → Like a class in OOP — defines the blueprint           │
│  → Stored in a registry (Docker Hub, ECR)                │
├──────────────────────────────────────────────────────────┤
│  CONTAINER                                               │
│  → A running instance of an image                        │
│  → Like an object instantiated from a class              │
│  → Isolated, has its own filesystem, network, processes  │
├──────────────────────────────────────────────────────────┤
│  REGISTRY                                                │
│  → A storage service for Docker images                   │
│  → Docker Hub: public registry (like npm for images)     │
│  → Amazon ECR: AWS's private registry (Day 27)           │
└──────────────────────────────────────────────────────────┘

Dockerfile ──(docker build)──► Image ──(docker run)──► Container
```

---

## 3. Installing Docker

```bash
# On Ubuntu (your EC2 instance or local Ubuntu machine)
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Allow running Docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
# Docker version 25.x.x
```

---

## 4. Writing Your First Dockerfile

Take your Node.js Express app from Day 6/7 and containerize it:

```
project/
  ├── index.js
  ├── package.json
  ├── package-lock.json
  └── Dockerfile          ← new file
```

```dockerfile
# Dockerfile

# ── Stage: choose the base image ──────────────────────────────
FROM node:20-alpine
# node:20-alpine = Node.js 20 on Alpine Linux (tiny ~5MB Linux distro)
# Alternative: node:20 (larger, Debian-based, more tools available)

# ── Set working directory inside the container ────────────────
WORKDIR /app
# All subsequent commands run from /app inside the container

# ── Copy dependency files FIRST (smart caching layer) ─────────
COPY package*.json ./
# Copying package.json separately from source code is intentional:
# If only your source changes, Docker reuses the cached npm install
# layer — making rebuilds much faster

# ── Install dependencies ───────────────────────────────────────
RUN npm install --production
# --production: skip devDependencies (testing tools, etc.)
# to keep the image lean

# ── Copy the rest of your application code ────────────────────
COPY . .
# Copies everything else (index.js, routes/, etc.)

# ── Tell Docker which port your app listens on ────────────────
EXPOSE 5000
# This is documentation only — doesn't actually open the port
# You open it with -p flag when running the container

# ── The command to start your app ─────────────────────────────
CMD ["node", "index.js"]
# CMD is the default command when the container starts
# Use array form (not string) to avoid shell interpretation issues
```

### The .dockerignore File — As Important as .gitignore

```bash
# .dockerignore
node_modules/     # never copy these — Docker installs fresh inside the image
.git/
.env              # never include secrets in images
*.log
.DS_Store
npm-debug.log
```

> **Critical:** If you forget `.dockerignore`, your local `node_modules/` gets copied into the image — potentially hundreds of MB of wrong-platform binaries, plus your local `.env` secrets baked into the image. Always create `.dockerignore` before `docker build`.

---

## 5. Building and Running Your Container

### Build the Image

```bash
# In your project directory (where Dockerfile lives)
docker build -t my-nodejs-app:1.0 .

# Breakdown:
#   docker build    → build command
#   -t              → tag (name:version)
#   my-nodejs-app   → image name
#   :1.0            → version tag (use "latest" if unspecified)
#   .               → build context (current directory)
```

```
Output shows Docker executing each Dockerfile instruction as a layer:
  Step 1/7 : FROM node:20-alpine
  Step 2/7 : WORKDIR /app
  Step 3/7 : COPY package*.json ./
  Step 4/7 : RUN npm install --production
  Step 5/7 : COPY . .
  Step 6/7 : EXPOSE 5000
  Step 7/7 : CMD ["node","index.js"]
  Successfully built abc123def456
  Successfully tagged my-nodejs-app:1.0
```

### Run the Container

```bash
docker run -d \
  -p 3000:5000 \
  --name my-running-app \
  --env-file .env \
  my-nodejs-app:1.0

# Breakdown:
#   -d              → detached mode (runs in background, like PM2)
#   -p 3000:5000    → map HOST port 3000 → CONTAINER port 5000
#   --name          → give the container a human-readable name
#   --env-file .env → load environment variables (safely, not baked in)
#   my-nodejs-app:1.0 → the image to run
```

```bash
# Test it
curl http://localhost:3000/health
# Should respond — your app is running in a container!
```

### Essential Docker Commands

```bash
docker ps                          # list running containers
docker ps -a                       # list all containers (including stopped)
docker logs my-running-app         # view stdout/stderr output
docker logs -f my-running-app      # follow logs live (like tail -f)
docker exec -it my-running-app sh  # open a shell INSIDE the container
docker stop my-running-app         # gracefully stop the container
docker rm my-running-app           # remove the stopped container
docker images                      # list all local images
docker rmi my-nodejs-app:1.0       # remove an image
docker image prune                 # remove all unused images (frees disk)
```

---

## 6. Docker Layers — Why Build Order Matters

Every Dockerfile instruction creates a **layer** — a cached snapshot of the filesystem at that point:

```
Layer 1: FROM node:20-alpine        ← pulled from Docker Hub (cached)
Layer 2: WORKDIR /app               ← tiny, instant
Layer 3: COPY package*.json ./      ← only changes if package.json changes
Layer 4: RUN npm install            ← EXPENSIVE: only reruns if layer 3 changed
Layer 5: COPY . .                   ← changes every time you edit code
Layer 6: EXPOSE 5000                ← instant
Layer 7: CMD ["node","index.js"]    ← instant
```

```
First build:  all 7 layers execute (slow — npm install runs)

You edit index.js and rebuild:
  Layers 1–4: CACHE HIT — skipped (nothing changed before COPY . .)
  Layer 5:    COPY . . — re-executes (your code changed)
  Layers 6–7: re-executes (tiny)
  Total: fast rebuild — no npm install re-run

You add a new npm package and rebuild:
  Layer 3: COPY package*.json — re-executes (package.json changed)
  Layer 4: RUN npm install    — re-executes (dependency of layer 3)
  Layers 5–7: re-execute
  Total: slower — npm install runs again
```

> **This is why `COPY package*.json ./` before `COPY . .` matters** — it ensures npm install only reruns when dependencies actually change, not on every code edit. This is the single most important Dockerfile optimization.

---

## 7. Multi-Stage Builds — Production-Grade Images

For a truly lean production image, use a **multi-stage build** — build in one stage, copy only the output to a clean final stage:

```dockerfile
# ── Stage 1: Builder ──────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install          # includes devDependencies for any build step
COPY . .
# If you had a TypeScript compile step: RUN npm run build

# ── Stage 2: Production ───────────────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app

# Copy only what's needed to run the app
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/index.js ./
# For TypeScript: COPY --from=builder /app/dist ./dist

# Security: don't run as root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 5000
CMD ["node", "index.js"]
```

```
Single-stage build size:  ~180 MB (includes build tools)
Multi-stage build size:   ~85 MB  (production essentials only)

Smaller image = faster pulls, faster deploys, smaller attack surface
```

---

## 8. Pushing to Amazon ECR — AWS's Private Registry

When you deploy to ECS (Day 27), your container images must be in **Amazon ECR (Elastic Container Registry)** — AWS's private Docker registry.

```bash
# Step 1: Create an ECR repository
aws ecr create-repository \
  --repository-name my-nodejs-app \
  --region ap-south-1

# Step 2: Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS \
  --password-stdin \
  YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com

# Step 3: Tag your image with the ECR URL
docker tag my-nodejs-app:1.0 \
  YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:1.0

# Step 4: Push to ECR
docker push \
  YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:1.0
```

```
Your image is now stored in ECR:
  ECR URL: YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/my-nodejs-app:1.0
  → ECS Fargate will pull this URL when launching containers (Day 27)
```

---

## 9. Docker Compose — Running Multiple Containers Locally

For local development where your app needs a database too:

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:5000"
    environment:
      - NODE_ENV=development
      - DB_HOST=postgres      # container name = hostname in Docker network
      - DB_PORT=5432
      - DB_NAME=appdb
      - DB_USER=postgres
      - DB_PASSWORD=password
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB:       appdb
      POSTGRES_USER:     postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data   # persist DB data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

```bash
# Start all services
docker compose up -d

# Your app is at http://localhost:3000
# PostgreSQL is at localhost:5432

# Stop everything
docker compose down

# Stop and delete volumes (fresh DB)
docker compose down -v
```

> Docker Compose is for **local development only** — in production on AWS you use ECS with RDS (not a containerized database). Never run your production database in a container without persistent storage — containers are ephemeral, data would be lost on restart.

---

## 10. Connecting Docker to Everything You've Learned

```
Days 4–7 (EC2 + PM2 + Nginx):
  Manual: SSH in, install Node, configure Nginx, run PM2
  Docker: build image once, ship everywhere — no manual setup

Day 14 (AMI):
  AMI = 20GB snapshot of an entire OS
  Docker image = 100MB of just your app + runtime
  Docker images are much faster to build, push, and pull

Day 19 (Auto Scaling):
  ASG uses AMIs to launch new instances (minutes)
  ECS uses Docker images to launch new containers (seconds)
  → Containers scale faster than VMs

Day 27 (ECS):
  ECS is AWS's container orchestration service
  It pulls your ECR image and runs it as tasks
  Everything you learn today directly enables Day 27
```

---

## 11. Interview Questions (Practice Explaining These Out Loud)

**Q1: What is Docker and what problem does it solve?**
> Docker packages an application and all its dependencies into a container — a portable, isolated unit that runs identically on any machine. It solves the "works on my machine" problem and makes deployments consistent, reproducible, and fast.

**Q2: What is the difference between a Docker image and a container?**
> An image is a read-only snapshot (the blueprint) built from a Dockerfile. A container is a running instance of that image — like a class vs an object in OOP.

**Q3: Why do you copy package.json before the rest of your source code in a Dockerfile?**
> Docker caches each layer. If you copy all files first, any code change invalidates the cache and forces npm install to rerun. By copying package.json first and running npm install as a separate layer, npm install only reruns when dependencies actually change — making rebuilds much faster.

**Q4: What is a multi-stage build and why use it?**
> A multi-stage build uses multiple FROM statements — one for building and one for the final production image. Only the necessary runtime files are copied to the final stage, resulting in a much smaller image without build tools, test dependencies, or intermediate files.

**Q5: What is Amazon ECR?**
> AWS's managed private Docker registry — where you store and manage Docker images for deployment to ECS, EKS, or any AWS container service. Like Docker Hub but private and integrated with IAM for access control.

**Q6: Should you run your production database in a Docker container?**
> No — containers are ephemeral. If a database container restarts, data in its writable layer is lost. Use a managed service like RDS instead, and only use Docker Compose with a database container locally for development convenience.

---

## 12. Hands-On Assignment for Today

1. **Install Docker** on your EC2 instance or local machine using the commands in Section 3.

2. **Containerize your Day 7 Express app:**
   - Write the `Dockerfile` from Section 4
   - Create `.dockerignore`
   - Run `docker build -t my-nodejs-app:1.0 .`

3. **Run the container** and test it:
   ```bash
   docker run -d -p 3000:5000 --name my-app my-nodejs-app:1.0
   curl http://localhost:3000/health
   ```

4. **Experiment with layers:**
   - Edit `index.js` → rebuild → notice layers 1–4 are cached
   - Add a new npm package → rebuild → notice npm install reruns

5. **Set up Docker Compose** for local development with PostgreSQL:
   - Create `docker-compose.yml` from Section 9
   - Run `docker compose up -d`
   - Connect to the local Postgres from inside the app container

6. **Push to ECR:**
   - Create an ECR repository
   - Tag and push your image
   - Verify it appears in ECR console

7. Write a short note covering:
   - The difference between an image and a container
   - Why layer order matters in a Dockerfile
   - Why you wouldn't run RDS in a Docker container in production

---

## 13. Day 26 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Docker packages my app and all its dependencies into a portable container image that runs identically everywhere. I write a Dockerfile with carefully ordered layers — copying package.json first so npm install only reruns when dependencies change. I build the image, tag it, and push it to Amazon ECR. For local development I use Docker Compose to run my app alongside a PostgreSQL container. In production, I use ECS to run containers from ECR images — never a containerized database, always RDS."*

---

**Next up — Day 27: ECS Overview (Container Orchestration on AWS)**
