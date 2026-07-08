# Module 19 — Dockerizing Microservices

> **Microservices Masterclass** | Level: Advanced | Track: Node.js Backend Engineering
> Prerequisite: Module 1–18 (especially Module 3 — Microservice Architecture)
> Next Module: Module 20 — Kubernetes for Microservices

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Learning Objectives](#2-learning-objectives)
3. [Problem Statement](#3-problem-statement)
4. [Why This Concept Exists](#4-why-this-concept-exists)
5. [Historical Background](#5-historical-background)
6. [Real-World Analogy](#6-real-world-analogy)
7. [Technical Definition](#7-technical-definition)
8. [Core Terminology](#8-core-terminology)
9. [Internal Working](#9-internal-working)
10. [Step-by-Step Request Flow](#10-step-by-step-request-flow)
11. [Architecture Overview](#11-architecture-overview)
12. [ASCII Diagrams](#12-ascii-diagrams)
13. [Mermaid Flowcharts](#13-mermaid-flowcharts)
14. [Mermaid Sequence Diagrams](#14-mermaid-sequence-diagrams)
15. [Component Diagrams](#15-component-diagrams)
16. [Deployment Diagrams](#16-deployment-diagrams)
17. [Database Interaction](#17-database-interaction)
18. [Failure Scenarios](#18-failure-scenarios)
19. [Scalability Discussion](#19-scalability-discussion)
20. [High Availability Considerations](#20-high-availability-considerations)
21. [CAP Theorem Implications](#21-cap-theorem-implications)
22. [Node.js Implementation](#22-nodejs-implementation)
23. [Express.js Examples](#23-expressjs-examples)
24. [Docker Examples](#24-docker-examples)
25. [Kafka/Redis Integration](#25-kafkaredis-integration)
26. [Error Handling](#26-error-handling)
27. [Logging & Monitoring](#27-logging--monitoring)
28. [Security Considerations](#28-security-considerations)
29. [Performance Optimization](#29-performance-optimization)
30. [Production Best Practices](#30-production-best-practices)
31. [Anti-Patterns and Common Mistakes](#31-anti-patterns-and-common-mistakes)
32. [Debugging Tips](#32-debugging-tips)
33. [Interview Questions](#33-interview-questions)
34. [Scenario-Based Questions](#34-scenario-based-questions)
35. [Hands-on Exercises](#35-hands-on-exercises)
36. [Mini Project](#36-mini-project)
37. [Advanced Project](#37-advanced-project)
38. [Summary](#38-summary)
39. [Revision Notes](#39-revision-notes)
40. [One-Page Cheat Sheet](#40-one-page-cheat-sheet)

---

## 1. Introduction

Throughout this masterclass, Docker and `docker-compose.yml` files have appeared in almost every module's code examples, treated as a familiar, assumed tool. This module is where we stop assuming and go deep: **what actually is a container, why does it solve microservices' deployment problem so well, and how do you write production-quality Dockerfiles and Compose configurations rather than "good enough for a demo" ones?**

Containers are the packaging technology that makes the entire microservices architectural style operationally realistic — without them (or something functionally equivalent), running dozens of independently-versioned services reliably across different environments would be a considerably harder, more error-prone endeavor, as it genuinely was in the pre-container era. This module gives you the depth to write efficient, secure, production-grade Docker configurations for Node.js microservices.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain what a container actually is, and how it differs from a virtual machine.
- Write efficient, secure, production-quality Dockerfiles for Node.js services, using multi-stage builds.
- Design Docker Compose configurations for realistic multi-service local development environments.
- Understand Docker networking and how services discover each other within Compose/Docker networks.
- Use volumes appropriately for persistent data and development workflows.
- Recognize and avoid common Docker anti-patterns that bloat images or introduce security risks.

---

## 3. Problem Statement

A team has 8 Node.js microservices, each developed by a different sub-team. Without deliberate Docker practices:

- One service's Dockerfile installs `devDependencies` in production, bloating the image to 1.2GB and slowing every deployment.
- Another service runs its application process as `root` inside the container — if that container is ever compromised, the attacker has root-level access within it, an avoidable, significant security risk.
- A third service's Dockerfile doesn't leverage layer caching correctly, meaning every single code change triggers a full `npm install` from scratch during the build, taking 5+ minutes even for a one-line change.
- Local development requires each engineer to manually start 8 separate `npm start` processes in 8 separate terminal tabs, remembering the correct order and environment variables for each — slow, error-prone, and inconsistent across the team.

This module solves each of these directly: efficient, secure, multi-stage Dockerfiles; a single `docker-compose up` command replacing 8 manual terminal tabs; and Docker networking making service-to-service communication within Compose trivially simple and consistent across every engineer's machine.

---

## 4. Why This Concept Exists

Docker (and containerization broadly) exists because **"it works on my machine" was, for decades, one of software engineering's most persistent, costly problems** — an application's behavior depends not just on its own code, but on the exact versions of its runtime, libraries, OS packages, and environment configuration, all of which could subtly differ between a developer's laptop, a staging server, and production. Containers solve this by packaging an application together with its **entire runtime environment** — the exact Node.js version, exact OS libraries, exact dependencies — into a single, portable, immutable artifact (the image) that behaves identically wherever it runs. For microservices specifically, this portability and consistency is what makes running dozens of independently-built, independently-versioned services operationally tractable at all — Module 3 introduced containers as one of microservices' essential building blocks precisely because of this.

---

## 5. Historical Background

- **2000s** — Various Linux kernel features (**cgroups**, introduced by Google engineers in 2006, and **namespaces**) provided the low-level building blocks for process isolation that containers are built on, but without a widely-adopted, easy-to-use packaging and distribution format around them.
- **2013** — **Docker** was released, providing a dramatically simpler developer experience on top of these existing Linux kernel features: a `Dockerfile` format for defining images, a registry system (**Docker Hub**) for sharing them, and simple CLI tooling — Docker's ease of use, more than any single new technical capability, is what triggered the explosion in container adoption.
- **2014** — **Docker Compose** was introduced, providing a simple, declarative way to define and run multi-container applications locally — directly solving the "8 manual terminal tabs" problem described in Section 3.
- **2015** — The **Open Container Initiative (OCI)** was formed, standardizing container image and runtime formats across the industry, ensuring images built with Docker could run correctly on other OCI-compliant runtimes (like containerd), reducing vendor lock-in concerns.
- **Present** — Docker remains the dominant containerization tool for local development and CI/CD image building, while **Kubernetes** (Module 20) has become the dominant **orchestrator** for running containers at production scale — Docker builds the box; Kubernetes decides where and how many of those boxes run.

---

## 6. Real-World Analogy

**Analogy: Standardized Shipping Containers (the technology's actual namesake)**

Before standardized shipping containers were introduced in the 1950s, cargo was loaded onto ships as loose, differently-shaped items — barrels, crates, sacks — each requiring custom handling, taking enormous time to load/unload, and often getting damaged or lost in the process. The invention of the **standardized shipping container** — a uniform, stackable box that any crane, truck, or ship could handle identically, regardless of what was actually inside it — revolutionized global shipping almost entirely through this **standardization of the outer packaging**, not through any change to the goods themselves.

Docker containers work identically for software: whatever is "inside" your service (Node.js, your dependencies, your code) is packaged into a **standardized, uniform box** (the container image) that any Docker-compatible host, anywhere — a developer's laptop, a CI/CD pipeline, a production Kubernetes cluster — can run identically, without needing to understand or specially handle what's inside. This standardization is precisely why containers solved the "it works on my machine" problem so effectively — the "machine" itself no longer matters, because the container carries its own consistent environment with it.

---

## 7. Technical Definition

> A **Container** is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system libraries, and settings — isolated from the host system and other containers via OS-level virtualization (Linux namespaces and cgroups), without the overhead of a full virtual machine's separate operating system kernel.

> A **Docker Image** is an immutable, layered filesystem snapshot built from a `Dockerfile`'s instructions, serving as the template from which running **containers** are instantiated.

> A **Multi-Stage Build** is a Dockerfile technique using multiple `FROM` statements, allowing you to use a full build environment (with compilers, dev dependencies) in an early stage, then copy only the final, necessary build artifacts into a much smaller, minimal final image — avoiding shipping build-time tools and dependencies into production.

> **Docker Compose** is a tool for defining and running multi-container Docker applications using a single declarative YAML file, handling networking, dependency ordering, and environment configuration for local development and testing.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Image** | An immutable, layered template from which containers are created |
| **Container** | A running (or stopped) instance of an image |
| **Dockerfile** | A text file of instructions defining how to build an image |
| **Layer** | Each instruction in a Dockerfile creates a cached, reusable filesystem layer |
| **Multi-Stage Build** | Using multiple build stages to keep the final image small and free of build-time-only tooling |
| **Docker Compose** | A tool/format for defining and running multi-container applications declaratively |
| **Volume** | A mechanism for persisting data outside a container's lifecycle, or sharing files between host and container |
| **Bridge Network** | Docker's default networking mode, allowing containers to communicate with each other by service/container name |
| **Base Image** | The starting image a Dockerfile's `FROM` instruction builds upon (e.g., `node:20-alpine`) |
| **Alpine** | A minimal Linux distribution commonly used as a lightweight Docker base image |

---

## 9. Internal Working

Here's how Docker's core mechanics work, and how they specifically benefit a multi-service Node.js system:

1. A `Dockerfile` describes, instruction by instruction, how to construct an image: starting from a **base image** (e.g., `node:20-alpine`), then copying files, installing dependencies, and specifying the startup command.
2. Each instruction in the Dockerfile creates a new, **cached layer** — if you rebuild the image and an earlier instruction (and everything it depends on) hasn't changed, Docker reuses the cached layer rather than re-executing it, dramatically speeding up repeated builds (this is why instruction **order** in a Dockerfile matters significantly, Section 22).
3. The built **image** is a read-only template; running it creates a **container** — an isolated process (or set of processes) with its own filesystem view, network namespace, and resource limits, but sharing the host machine's kernel (unlike a full virtual machine, which runs its own separate kernel — this is why containers are dramatically lighter-weight and faster to start than VMs).
4. **Docker Compose** reads a `docker-compose.yml` file describing multiple services, automatically creating a shared **bridge network** so containers can reach each other by their service name (e.g., `order-service` can reach `http://payment-service:4003` simply because Compose configured DNS resolution for that name within the shared network) — directly implementing the Service Discovery concepts from Module 11 in a local development context.
5. **Volumes** allow data to persist beyond a container's lifecycle (e.g., a database's actual data files) or allow local source code to be mounted into a running container for rapid development iteration without rebuilding the image on every change.

---

## 10. Step-by-Step Request Flow

**Scenario: Building and running a multi-service Node.js system locally with Docker Compose.**

```
Step 1:  Engineer runs `docker-compose up --build`
Step 2:  Docker Compose reads docker-compose.yml, identifying all
         defined services (order-service, payment-service, order-db, kafka)
Step 3:  For each service with a `build` directive, Docker builds
         the image from its Dockerfile, using MULTI-STAGE builds
         (Section 22) to keep final images small
Step 4:  Docker Compose creates a shared bridge NETWORK for all
         these services
Step 5:  Docker Compose starts containers in DEPENDENCY ORDER
         (respecting `depends_on`), e.g., order-db and kafka before
         order-service, which needs them
Step 6:  order-service's container starts; it can reach payment-service
         simply via the hostname "payment-service" (Docker's internal
         DNS resolves this to the correct container's IP, WITHIN
         the shared network) - exactly Module 11's discovery pattern
Step 7:  Volumes (if configured) mount persistent data directories
         (e.g., order-db's actual data files) so data survives
         container restarts
Step 8:  The engineer can now test the FULL multi-service system
         locally, with realistic inter-service networking, using
         ONE command instead of manually managing 8 terminal tabs
```

---

## 11. Architecture Overview

```
                     Docker Compose Bridge Network
        ┌─────────────────────────────────────────────┐
        │                                                 │
        │  order-service ──────▶ payment-service            │
        │  (container)           (container)                 │
        │       │                     │                       │
        │       ▼                     ▼                       │
        │  order-db               payment-db                   │
        │  (container +           (container +                  │
        │   volume for            volume for                     │
        │   data persistence)      data persistence)               │
        │       │                                                │
        │       ▼                                                │
        │   kafka (container)                                     │
        │                                                 │
        └─────────────────────────────────────────────┘
                     All reachable by SERVICE NAME
                (order-service, payment-service, etc.)
                  via Docker's internal DNS resolution
```

---

## 12. ASCII Diagrams

### 12.1 Container vs Virtual Machine

```
VIRTUAL MACHINE (heavier):

  Host OS
     │
  Hypervisor
     │
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │ Guest OS  │  │ Guest OS  │  │ Guest OS  │  <- EACH VM has its OWN
  │ (full)     │  │ (full)     │  │ (full)     │     full operating system
  │  App A     │  │  App B     │  │  App C     │     kernel - heavy, slow
  └─────────┘  └─────────┘  └─────────┘     to start


CONTAINER (lighter):

  Host OS (ONE shared kernel)
     │
  Container Runtime (Docker)
     │
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │  App A     │  │  App B     │  │  App C     │  <- containers SHARE the
  │ (isolated   │  │ (isolated   │  │ (isolated   │     host's kernel - much
  │  process)   │  │  process)   │  │  process)   │     lighter, faster to start
  └─────────┘  └─────────┘  └─────────┘
```

### 12.2 Multi-Stage Build

```
STAGE 1 (builder - has ALL dev tools, dependencies):

  FROM node:20 AS builder
  COPY package*.json ./
  RUN npm ci                    <- includes DEV dependencies
  COPY . .
  RUN npm run build             <- e.g., TypeScript compilation


STAGE 2 (final - ONLY what's needed to RUN, nothing else):

  FROM node:20-alpine
  COPY --from=builder /app/dist ./dist       <- ONLY compiled output
  COPY --from=builder /app/node_modules ./node_modules  <- production deps
  CMD ["node", "dist/app.js"]

  RESULT: final image does NOT contain TypeScript compiler,
  dev dependencies, or source files - just what's needed to RUN,
  dramatically smaller and more secure
```

### 12.3 Docker Layer Caching

```
Dockerfile instruction order MATTERS for cache efficiency:

  GOOD (dependencies rarely change, code changes often):

    COPY package*.json ./     <- Layer 1: cached unless package.json changes
    RUN npm ci                 <- Layer 2: cached unless Layer 1 changed
    COPY . .                   <- Layer 3: changes on EVERY code edit
    (only Layer 3 needs rebuilding on a typical code change - FAST)


  BAD (dependencies re-installed on EVERY code change):

    COPY . .                   <- Layer 1: changes on EVERY code edit
    RUN npm ci                 <- Layer 2: re-runs EVERY time, since
                                   Layer 1 (which it depends on) changed
    (npm ci re-executes on EVERY build, even for a one-line
     code change - SLOW, wastes minutes per build)
```

---

## 13. Mermaid Flowcharts

### 13.1 Docker Build and Run Flow

```mermaid
flowchart TD
    A[Dockerfile] --> B[docker build]
    B --> C[Layer 1: Base image]
    C --> D[Layer 2: Copy package.json, install deps]
    D --> E[Layer 3: Copy source code]
    E --> F[Final Image]
    F --> G[docker run / docker-compose up]
    G --> H[Running Container]
```

### 13.2 Multi-Stage Build Decision

```mermaid
flowchart TD
    A[Does your service need a build step?] --> B{TypeScript, bundling, or other compilation?}
    B -- Yes --> C[Use Multi-Stage Build - builder stage + slim final stage]
    B -- No, plain JavaScript --> D{Do you still want to minimize final image size?}
    D -- Yes --> E[Still consider a slim final stage with only prod dependencies]
    D -- No --> F[Single-stage Dockerfile may be sufficient]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Docker Compose Startup Sequence

```mermaid
sequenceDiagram
    participant Dev as Engineer
    participant DC as Docker Compose
    participant DB as order-db container
    participant K as kafka container
    participant O as order-service container

    Dev->>DC: docker-compose up --build
    DC->>DC: Build images (multi-stage builds)
    DC->>DC: Create shared bridge network
    DC->>DB: Start (no dependencies)
    DC->>K: Start (no dependencies)
    DC->>O: Start (depends_on: order-db, kafka)
    O->>DB: Connect via hostname "order-db"
    O->>K: Connect via hostname "kafka"
    Note over O,K: Docker's internal DNS resolves service<br/>names to container IPs automatically
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                   docker-compose.yml                        │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐      │
│  │ order-service     │ │ payment-service   │ │ order-db          │      │
│  │ (build: ./order-  │ │ (build: ./payment- │ │ (image: postgres)   │      │
│  │  service)          │ │  service)           │ │                      │      │
│  └───────────────┘ └───────────────┘ └───────────────┘      │
│  ┌───────────────┐                                          │
│  │ kafka              │                                          │
│  │ (image: bitnami/    │                                          │
│  │  kafka)              │                                          │
│  └───────────────┘                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
LOCAL DEVELOPMENT (Docker Compose):

  Single machine, single `docker-compose up` command,
  ONE shared bridge network, ALL services running as
  sibling containers


PRODUCTION (typically Kubernetes, Module 20):

  Docker images (the SAME images built for local dev/CI)
  are pushed to a CONTAINER REGISTRY, then pulled and run
  by Kubernetes across a MULTI-NODE cluster, with
  Kubernetes handling scaling, healing, and networking
  at a much larger scale than Compose is designed for

  KEY INSIGHT: the IMAGE itself is often IDENTICAL between
  local dev and production - only the ORCHESTRATION differs
  (Compose locally, Kubernetes in production)
```

---

## 17. Database Interaction

Docker Compose commonly runs a service's database as a sibling container for local development, using **volumes** to persist data across restarts:

```yaml
services:
  order-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=orders
    volumes:
      - order-db-data:/var/lib/postgresql/data  # NAMED VOLUME - persists data

volumes:
  order-db-data:  # Docker manages this volume's actual storage location
```

Without this volume, stopping and removing the `order-db` container would **permanently lose all its data** — a critical distinction for local development workflows where you want your test data to survive routine `docker-compose down`/`up` cycles.

---

## 18. Failure Scenarios

| Scenario | Docker/Compose Handling |
|---|---|
| A container crashes | Docker can be configured with a `restart` policy (e.g., `restart: unless-stopped`) to automatically restart it |
| The Dockerfile doesn't specify a non-root user | The container's process runs as `root` by default — a security risk if the container is ever compromised (Section 28) |
| A code change requires a full rebuild every time (slow) | Poor Dockerfile layer ordering (Section 12.3) is almost always the cause — reorder instructions so dependency installation is cached separately from source code copying |
| Two services can't reach each other by name | Usually caused by them being on different Docker networks, or a typo in the service name used as the hostname — verify both are defined in the SAME `docker-compose.yml` (or explicitly shared network) |
| Data disappears after `docker-compose down` | No volume was configured for a stateful service (like a database) — `docker-compose down` (without `-v`) preserves named volumes, but a container without ANY volume configured loses its data entirely on removal |

---

## 19. Scalability Discussion

Docker Compose is explicitly a **local development and simple testing tool**, not designed for production-scale orchestration — it lacks built-in support for multi-host clustering, automatic healing across machine failures, and sophisticated scheduling that a system like Kubernetes (Module 20) provides. For local development, Compose's `scale` option or `deploy.replicas` (in newer Compose versions) can spin up multiple instances of a service for basic testing, but genuine production scaling, load balancing across multiple physical hosts, and self-healing require a dedicated orchestrator, which is precisely why Module 20 exists as this module's natural next step.

---

## 20. High Availability Considerations

Docker Compose, again, is not intended to provide production-grade high availability — it typically runs on a single host, meaning that host's failure takes down every container running on it. For genuine HA, you need Kubernetes (or a similar orchestrator) distributing containers across multiple physical machines, with automatic rescheduling upon node failure — concepts introduced in Module 3 and covered in depth in Module 20. Docker Compose's `restart` policies provide only single-host process-level resilience, not machine-level or availability-zone-level resilience.

---

## 21. CAP Theorem Implications

Docker and Docker Compose themselves are packaging/orchestration tools and don't directly make CAP theorem trade-offs — those trade-offs belong to the actual services and databases running inside the containers (as extensively covered in earlier modules). However, it's worth noting that a database run as a **single Compose container** (as is common in local development) has none of the replication or distributed consensus mechanisms discussed in Module 11 and elsewhere — local Compose setups are appropriately simplified for development, and should never be mistaken for a production-representative topology from a CAP or HA perspective.

---

## 22. Node.js Implementation

Let's build a production-quality, multi-stage Dockerfile for a Node.js service, contrasting it with a naive, unoptimized version.

**Naive Dockerfile (anti-pattern, for contrast):**
```dockerfile
FROM node:20
WORKDIR /app
COPY . .                 # Copies EVERYTHING first, including source code
RUN npm install           # Re-runs on EVERY code change (poor caching)
EXPOSE 4002
CMD ["node", "src/app.js"]
# Problems: full node:20 image (not slim), runs as root, includes
# devDependencies, poor layer caching, no .dockerignore consideration
```

**Production-quality, multi-stage Dockerfile:**
```dockerfile
# ---- Stage 1: Build ----
FROM node:20-alpine AS builder
WORKDIR /app

# Copy ONLY package files first - this layer is cached unless
# dependencies actually change, dramatically speeding up rebuilds
COPY package*.json ./
RUN npm ci  # installs ALL dependencies, including dev, needed for build

# Now copy source code (changes often, but doesn't invalidate the
# dependency installation layer above)
COPY . .

# If this were TypeScript, this is where compilation would happen:
# RUN npm run build

# ---- Stage 2: Production ----
FROM node:20-alpine AS production
WORKDIR /app

# Install ONLY production dependencies in the FINAL image
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy ONLY the application code needed to run (not dev tools,
# test files, or build artifacts not needed at runtime)
COPY --from=builder /app/src ./src

# Create and use a NON-ROOT user for security (Section 28)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 4002

# Use the array form (exec form) for CMD - ensures proper signal
# handling (e.g., SIGTERM for graceful shutdown, Module 11)
CMD ["node", "src/app.js"]
```

**`.dockerignore`** (critical, often-forgotten companion file):
```
node_modules
npm-debug.log
.git
.env
.env.local
*.md
tests/
coverage/
.dockerignore
Dockerfile
```

---

## 23. Express.js Examples

The Express application code itself doesn't change based on containerization, but it's worth ensuring it handles **graceful shutdown** correctly, since Docker sends a `SIGTERM` signal when stopping a container:

```javascript
import express from "express";

const app = express();
// ... routes ...

const server = app.listen(4002, () => {
  console.log("Order Service running on port 4002");
});

// Graceful shutdown: Docker (and Kubernetes, Module 20) send SIGTERM
// before forcibly killing a container - handle it to finish in-flight
// requests and close connections cleanly, rather than dropping them abruptly
process.on("SIGTERM", () => {
  console.log("SIGTERM received, shutting down gracefully...");
  server.close(() => {
    console.log("HTTP server closed");
    process.exit(0);
  });
});
```

---

## 24. Docker Examples

A realistic, production-quality `docker-compose.yml` for local development of a multi-service system:

```yaml
version: "3.9"

services:
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "4002:4002"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - PAYMENT_SERVICE_URL=http://payment-service:4003
      - KAFKA_BROKER=kafka:9092
    depends_on:
      order-db:
        condition: service_healthy   # waits for a HEALTHY db, not just "started"
      kafka:
        condition: service_started
    volumes:
      - ./order-service/src:/app/src   # mount source for live-reload in DEV only
    restart: unless-stopped

  payment-service:
    build: ./payment-service
    ports: ["4003:4003"]
    environment:
      - DATABASE_URL=postgresql://user:pass@payment-db:5432/payments
    depends_on: [payment-db]
    restart: unless-stopped

  order-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - order-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d orders"]
      interval: 5s
      timeout: 3s
      retries: 5

  payment-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=payments
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - payment-db-data:/var/lib/postgresql/data

  kafka:
    image: bitnami/kafka:latest
    ports: ["9092:9092"]

volumes:
  order-db-data:
  payment-db-data:
```

Note the `healthcheck` on `order-db` and the corresponding `condition: service_healthy` on `order-service` — this ensures `order-service` doesn't start attempting database connections before PostgreSQL is actually ready to accept them, a common source of flaky local startup ordering.

---

## 25. Kafka/Redis Integration

Both Kafka and Redis are commonly run as simple sibling containers in local Compose setups, exactly as shown throughout this masterclass's Docker examples in every prior module. For local development, using an official or well-maintained community image (`bitnami/kafka`, `redis:7-alpine`) is standard practice, with production deployments (Module 20) typically using a more sophisticated, purpose-built deployment (e.g., a managed Kafka service, or a dedicated Kafka Kubernetes Operator) rather than a simple container run the same way as local development.

---

## 26. Error Handling

Handle build-time failures clearly, and ensure runtime errors inside a container are visible via standard output (which Docker captures as container logs):

```javascript
// Ensure UNHANDLED errors are logged clearly and cause a clean exit,
// so Docker/Kubernetes can detect the failure and restart appropriately,
// rather than the process hanging in a broken, unresponsive state
process.on("uncaughtException", (err) => {
  console.error("FATAL: Uncaught exception:", err);
  process.exit(1);
});

process.on("unhandledRejection", (reason) => {
  console.error("FATAL: Unhandled promise rejection:", reason);
  process.exit(1);
});
```

---

## 27. Logging & Monitoring

- Node.js applications inside containers should log to **stdout/stderr** (not to a file inside the container) — Docker automatically captures stdout/stderr as the container's logs, accessible via `docker logs <container>` and easily forwarded to centralized logging systems in production.
- Use **structured JSON logging** (as established in earlier modules) so container log aggregation tools can parse and index log fields correctly.
- Monitor container resource usage (`docker stats` locally, or cluster-level metrics in production) to catch a service consuming unexpectedly high CPU/memory, which might indicate a resource leak or a missing Bulkhead/resource limit (Module 18).

---

## 28. Security Considerations

- **Never run application processes as `root`** inside a container — always create and switch to a dedicated non-root user (Section 22's `USER appuser` line), limiting the impact of a potential container compromise.
- **Never bake secrets into a Docker image** — an image's layers are inspectable, and a secret baked into any layer (even if later "removed" in a subsequent layer) can often still be extracted; always inject secrets at runtime (Module 12's secrets management) via environment variables, mounted secret files, or a runtime secrets client, never via `COPY` or `ARG`/`ENV` with a hardcoded value.
- Use minimal base images (`node:20-alpine` rather than `node:20`) to reduce the attack surface — fewer installed packages means fewer potential vulnerabilities.
- Regularly rebuild images to pick up base image security patches, and use vulnerability scanning tools (e.g., `docker scan`, Trivy, Snyk) as part of your CI/CD pipeline.

---

## 29. Performance Optimization

- Order Dockerfile instructions to **maximize layer cache reuse** (Section 12.3) — copy dependency manifests and install dependencies before copying the rest of the source code.
- Use **multi-stage builds** (Section 22) to keep final images small, which speeds up image pull times, deployment rollouts, and reduces storage/bandwidth costs across your infrastructure.
- Use a `.dockerignore` file (Section 22) to prevent unnecessary files (`node_modules`, `.git`, test files) from being sent to the Docker build context at all, speeding up build times and preventing accidental inclusion in the image.
- Consider using `npm ci` instead of `npm install` in Dockerfiles — `npm ci` is faster, deterministic (uses `package-lock.json` exactly), and better suited for reproducible, automated builds.

---

## 30. Production Best Practices

- Always use multi-stage builds and non-root users, as demonstrated in Section 22, as a non-negotiable team-wide standard.
- Tag images with meaningful, immutable identifiers (e.g., a Git commit SHA or a semantic version) rather than relying solely on `latest`, ensuring reproducible deployments and easy rollbacks.
- Set explicit resource limits (`mem_limit`, `cpus` in Compose; resource requests/limits in Kubernetes, Module 20) so a single misbehaving container can't consume unbounded host resources.
- Include a proper **health check** (`HEALTHCHECK` instruction in the Dockerfile, or Compose's `healthcheck`) so orchestrators can correctly determine container readiness, directly connecting to Module 11's readiness probe concepts.
- Keep a consistent Dockerfile structure/pattern across all your organization's services, easing maintenance and onboarding.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Running as root inside the container** | Unnecessarily increases the security impact of a potential container compromise |
| **Copying source code before installing dependencies** | Destroys layer caching, causing full dependency reinstalls on every code change |
| **No `.dockerignore` file** | Sends unnecessary files (node_modules, .git) to the build context, slowing builds and risking accidental secret inclusion |
| **Baking secrets into image layers** | Secrets become extractable from the image itself, even if "removed" in a later layer |
| **Using a full, non-slim base image unnecessarily** | Bloats image size and increases the security attack surface with unneeded packages |
| **No non-root user, no resource limits, no health checks** | Collectively signal an image not yet ready for genuine production use |

```
Copying source before installing dependencies (anti-pattern):

  COPY . .              <- invalidates cache on EVERY file change
  RUN npm ci             <- re-runs on EVERY build, even for a
                            one-line README.md change

  FIX: COPY package*.json first, RUN npm ci, THEN COPY the rest
  - dependencies only reinstall when package.json ACTUALLY changes
```

---

## 32. Debugging Tips

- Use `docker build --progress=plain` to see detailed build output when diagnosing a slow or failing build.
- Use `docker exec -it <container> sh` to get an interactive shell inside a running container for direct debugging.
- If a container exits immediately, check `docker logs <container>` first — this almost always reveals the root cause (a startup error, missing environment variable, etc.).
- If layer caching doesn't seem to be working as expected, use `docker history <image>` to inspect each layer's size and understand which instructions are triggering rebuilds.
- If two containers can't reach each other, verify they're on the SAME Docker network (`docker network inspect <network>` lists connected containers) — a common oversight when services are defined across multiple separate Compose files.

---

## 33. Interview Questions

### Easy
1. What is a Docker container, and how does it differ from a virtual machine?
2. What is a Dockerfile?
3. What is Docker Compose used for?
4. Why should you avoid running application processes as root inside a container?
5. What is the purpose of a `.dockerignore` file?

### Medium
6. Explain Docker layer caching and why Dockerfile instruction order matters.
7. What is a multi-stage build, and what problem does it solve?
8. How does Docker Compose enable service discovery between containers?
9. Why should secrets never be baked into a Docker image's layers?
10. What is the difference between the local Docker Compose environment and a production Kubernetes deployment?

### Hard
11. Design a production-quality, multi-stage Dockerfile for a Node.js TypeScript service, explaining each instruction's purpose.
12. How would you diagnose and fix a Dockerfile causing unnecessarily slow rebuilds on every minor code change?
13. Explain the security implications of running a container as root, and how a non-root user configuration mitigates them.
14. Discuss the trade-offs of using `node:20-alpine` versus `node:20` (the full Debian-based image) as a base image.
15. Design a local Docker Compose environment for a 5-service system with proper health checks, dependency ordering, and data persistence for two databases.

---

## 34. Scenario-Based Questions

1. Your team's CI/CD pipeline takes 8 minutes to build a Docker image for even a one-line code change. Diagnose the likely cause and propose a fix.
2. A security audit flags that your production containers are running as root. What's your remediation plan, and what risks does this address?
3. An engineer accidentally committed a database password directly into a Dockerfile's `ENV` instruction 3 months ago. What's your incident response, given how Docker image layers work?
4. Your local Docker Compose setup works fine, but a new engineer's `docker-compose up` fails because `order-service` tries to connect to `order-db` before it's ready. How would you fix this using Compose's health check features?
5. Leadership wants to know why your Docker images are 1.2GB when a similar open-source project's images are 150MB. What would you investigate and likely find?

---

## 35. Hands-on Exercises

1. Write a naive, single-stage Dockerfile for a simple Express service, then refactor it into a multi-stage, non-root, cache-optimized version, and compare the resulting image sizes.
2. Write a `.dockerignore` file for a typical Node.js project, and explain what each excluded pattern prevents.
3. Build a `docker-compose.yml` for a 3-service system (two Node.js services + one PostgreSQL database) with proper health checks and dependency ordering.
4. Use `docker history` on an image you've built to identify which layer contributes the most to its total size, and propose an optimization.
5. Intentionally introduce a Dockerfile ordering mistake (copying source before installing dependencies), measure the rebuild time after a small code change, then fix it and measure again.

---

## 36. Mini Project

**Build: A Production-Quality Dockerized Service**

1. Build a simple Express service and write a production-quality, multi-stage Dockerfile for it, including a non-root user, proper layer caching, and a `.dockerignore` file.
2. Add a `HEALTHCHECK` instruction (or Compose `healthcheck`) hitting a `/health` endpoint.
3. Build the image and verify: (a) it runs as a non-root user (`docker exec <container> whoami`), (b) rebuilding after a source-only code change is fast (cached dependency layer), and (c) the final image size is reasonably small (compare against a naive single-stage version).

---

## 37. Advanced Project

**Build: A Full Multi-Service Docker Compose Environment**

1. Build a 4-service system (`order-service`, `payment-service`, `order-db`, `payment-db`, plus `kafka`) with production-quality, multi-stage Dockerfiles for both Node.js services.
2. Configure `docker-compose.yml` with proper health checks for both databases, correct `depends_on` conditions, named volumes for data persistence, and a shared bridge network.
3. Demonstrate service discovery: `order-service` successfully reaches `payment-service` and `kafka` purely by their Compose service names, with no hardcoded IPs anywhere.
4. Verify graceful shutdown: send a `SIGTERM` to `order-service`'s container (`docker stop`) and confirm, via logs, that it shuts down cleanly rather than being abruptly killed.
5. Write a short document comparing this Compose setup to what a production Kubernetes deployment (Module 20) of the same system would look like, identifying what stays the same (the images themselves) and what differs (orchestration, scaling, health management).

---

## 38. Summary

- Docker containers package an application with its complete runtime environment, providing consistent, portable behavior across development, testing, and production — directly solving the "it works on my machine" problem.
- Multi-stage Dockerfiles keep production images small and secure by separating build-time tooling from the final runtime image.
- Dockerfile instruction order directly affects build speed via layer caching — dependencies should be installed before source code is copied.
- Docker Compose provides local, declarative multi-container orchestration, including automatic service discovery by container/service name, ideal for development but not intended for production-scale HA or multi-host scaling.
- Security best practices (non-root users, no baked-in secrets, minimal base images) are non-negotiable for any production-bound container.

---

## 39. Revision Notes

- Container: isolated process sharing the host kernel (lighter than a VM, which has its own kernel).
- Multi-stage build: builder stage (full tooling) + slim final stage (only runtime essentials).
- Layer caching: order Dockerfile instructions so dependency installation is cached separately from source code changes.
- Docker Compose: local multi-container orchestration + automatic service-name-based networking.
- Non-root user + no baked-in secrets + minimal base image = baseline production security requirements.
- Docker Compose ≠ production orchestration — that's Kubernetes's job (Module 20).

---

## 40. One-Page Cheat Sheet

```
CONTAINER:            isolated process, SHARES host kernel (lighter than a VM)
IMAGE:                immutable, layered template a container is created from
MULTI-STAGE BUILD:     builder stage (full tools) -> slim final stage (runtime only)
LAYER CACHING:         order matters! deps BEFORE source code copy
DOCKER COMPOSE:        local multi-container orchestration + automatic service DNS
VOLUME:                persists data beyond a container's lifecycle

GOLDEN RULES:
  - ALWAYS use multi-stage builds to keep final images small
  - ALWAYS run as a NON-ROOT user inside the container
  - NEVER bake secrets into image layers - inject at runtime instead
  - Order Dockerfile instructions for MAXIMUM layer cache reuse
  - Use a `.dockerignore` file - always
  - Docker Compose is for LOCAL DEV - production needs a real orchestrator (K8s)
```

---

**Suggested Next Module:** Module 20 — Kubernetes for Microservices (Pods, Deployments, Services, ConfigMaps, Secrets, and Ingress for running microservices at production scale)
