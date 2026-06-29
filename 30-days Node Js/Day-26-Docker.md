# Day 26 — Docker: Containerization, Dockerfiles, Multi-Stage Builds, docker-compose

> **Why this day matters:** Almost every real backend job today deploys Node apps via containers. Beyond "I know the docker commands," interviewers want to know you understand WHAT a container actually is (vs. a VM — a very commonly confused pair), why multi-stage builds matter for production image size/security, and how docker-compose lets you run your whole local stack (app + Postgres + Redis) with one command, instead of manually installing and managing each service.

---

## 1. Background: what problem does Docker actually solve?

### The "works on my machine" problem

Before containers, a genuinely common, frustrating problem was: an app works perfectly on a developer's laptop, but breaks in production (or on a teammate's machine) because of subtle environment differences — a different Node version, a missing system library, a different OS entirely. **Docker solves this by packaging your application TOGETHER WITH everything it needs to run** (the exact Node version, OS-level dependencies, your code, your `node_modules`) into a single, portable unit called an **image**, which runs identically wherever Docker itself is installed — your laptop, a teammate's laptop, a CI pipeline, or a production server.

### Containers vs Virtual Machines — THE distinction interviewers expect precisely

**This is one of the most commonly asked Docker fundamentals questions, and a lot of engineers explain it vaguely. Be precise:**

```
VIRTUAL MACHINE:                    CONTAINER:

┌─────────────────────┐            ┌─────────────────────┐
│   App A   │  App B    │            │   App A   │  App B    │
├───────────┼───────────┤            ├───────────┼───────────┤
│  Guest OS │  Guest OS  │            │  (no guest OS --     │
│  (full!)  │  (full!)   │            │   just the app's      │
├───────────┴───────────┤            │   dependencies)        │
│      Hypervisor         │            ├─────────────────────┤
├─────────────────────┤            │   Docker Engine          │
│      Host OS             │            ├─────────────────────┤
└─────────────────────┘            │      Host OS (shared)    │
                                     └─────────────────────┘
```

**The precise distinction:** a VM virtualizes an entire computer, including a FULL guest operating system, for each instance — heavy (gigabytes), slow to start (minutes), but fully isolated at the OS level. A **container virtualizes at the OS level** — all containers on one machine SHARE the host machine's kernel, and a container just packages the application and its specific dependencies (libraries, runtime), NOT a full separate OS. This makes containers dramatically lighter (megabytes, not gigabytes) and faster to start (seconds, not minutes), at the cost of slightly less isolation than a full VM (since the kernel is shared).

**Interview tip — the one-sentence version to have ready:** *"A VM virtualizes hardware and includes a full guest OS per instance; a container virtualizes at the OS level, sharing the host's kernel, making containers far lighter and faster to start, while still isolating the application's filesystem, processes, and dependencies."*

---

## 2. Writing a Dockerfile for a Node.js App

### Background: what is a Dockerfile, precisely?

A **Dockerfile** is a text file containing step-by-step INSTRUCTIONS for building a Docker **image** (a static, reusable blueprint). When you run a container, you're running a live INSTANCE of that image — same relationship as a class and an object in OOP terms, if that helps make it click.

```dockerfile
# Dockerfile

# FROM specifies the BASE IMAGE to build on top of -- you're not
# starting from nothing, you're starting from an existing image that
# already has Node.js installed. 'alpine' is a minimal Linux
# distribution, chosen here specifically to keep the final image small.
FROM node:20-alpine

# WORKDIR sets the working directory INSIDE the container for all
# subsequent instructions -- like running 'cd /app' before everything else
WORKDIR /app

# COPY package.json and package-lock.json FIRST, before copying the
# rest of your code -- this is a DELIBERATE optimization, not an
# accident, explained in detail in the layer caching section below
COPY package*.json ./

# Installing dependencies INSIDE the container -- this runs AS PART OF
# building the image, so the resulting image already has node_modules
# baked in, ready to run immediately when a container starts from it
RUN npm install --production
# '--production' skips devDependencies (Day 1's distinction!) -- you
# don't need testing/linting tools INSIDE your production container

# NOW copy the rest of your application code
COPY . .

# EXPOSE documents which port the app listens on -- this is purely
# INFORMATIONAL/documentation by itself; it does NOT actually publish
# the port (that's done with the '-p' flag when RUNNING the container,
# covered below) -- a commonly misunderstood detail
EXPOSE 3000

# CMD specifies the command to run when a CONTAINER starts from this
# image -- this is the actual "start the app" command, different from
# RUN (which executes during the BUILD process, not at container startup)
CMD ["node", "app.js"]
```

```bash
# Building an image FROM the Dockerfile -- creates a reusable blueprint
docker build -t my-node-app:1.0 .
# '-t' tags the image with a name and version; '.' means "use the
# Dockerfile in the current directory, and use this directory as the
# build context (what COPY can access)"

# Running a CONTAINER (a live instance) from that image
docker run -p 3000:3000 my-node-app:1.0
# '-p 3000:3000' maps port 3000 ON YOUR MACHINE to port 3000 INSIDE the
# container -- THIS is what actually makes the app reachable from
# outside the container, distinct from the Dockerfile's EXPOSE instruction

# Running in the background (detached) and naming it for easy reference
docker run -d -p 3000:3000 --name my-app-container my-node-app:1.0

docker ps                    # list running containers
docker logs my-app-container # view a container's logs
docker stop my-app-container
```

### Layer Caching — why instruction ORDER in a Dockerfile genuinely matters

**Background: this is the detail that separates someone who copy-pasted a Dockerfile from someone who understands it.** Docker builds an image in LAYERS, one per instruction, and CACHES each layer. If you re-build an image and an EARLIER layer hasn't changed, Docker reuses the cached layer instead of re-running that step — but the moment ONE layer changes, EVERY layer after it must be rebuilt too, even if their own content didn't change.

```dockerfile
# THE WRONG ORDER (a real, common mistake):
FROM node:20-alpine
WORKDIR /app
COPY . .                     # copies EVERYTHING, including package.json AND your source code
RUN npm install --production # this layer is now INVALIDATED every single time you change
                               # ANY source file, even a tiny one-line change to a route handler --
                               # forcing a full npm install to re-run on EVERY rebuild, which is slow

# THE RIGHT ORDER (shown in the Dockerfile above, repeated here for contrast):
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./         # ONLY copies package.json/lock files first
RUN npm install --production  # this layer is CACHED and SKIPPED on rebuilds, as long as
                                # package.json/package-lock.json haven't changed --
                                # even if your source code changes constantly
COPY . .                       # source code copied LAST -- changes here don't invalidate
                                # the (expensive) npm install layer above it
```

**Interview tip:** if asked "why does the order of instructions in a Dockerfile matter," this layer caching explanation, with the specific `package.json`-before-source-code example, is exactly the depth expected — it's a genuinely practical detail that affects real build times in CI/CD pipelines (Day 27).

---

## 3. Multi-Stage Builds — keeping production images small and secure

### Background: what problem does this solve?

A single-stage Dockerfile (as shown above) bundles EVERYTHING needed to BUILD your app (potentially including devDependencies, build tools, TypeScript compilers, test files) into the SAME final image that runs in production. This makes the image larger than necessary, and includes tools/files that have no business being in a production container (more attack surface, more to patch/secure, slower to pull/deploy).

```dockerfile
# Multi-stage build for a TypeScript Node.js app (a very common real
# scenario -- TypeScript needs to be COMPILED to JS before running,
# and the compiler itself isn't needed once that's done)

# ---- STAGE 1: "builder" -- has everything needed to BUILD the app ----
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm install              # installs ALL dependencies, including devDependencies
                               # (needed here, since the TypeScript compiler is a devDependency)

COPY . .
RUN npm run build             # compiles TypeScript to JavaScript, e.g. into a /dist folder

# ---- STAGE 2: the FINAL, production image -- starts FRESH ----
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --production  # ONLY production dependencies this time

# COPY --from=builder pulls ONLY the specific files we actually need
# (the compiled output) FROM the previous stage, leaving behind the
# TypeScript compiler, dev dependencies, raw .ts source files, and
# anything else that stage 1 needed but stage 2 doesn't
COPY --from=builder /app/dist ./dist

EXPOSE 3000
CMD ["node", "dist/app.js"]
```

**Why this matters concretely:** the FINAL image only contains what's needed to actually RUN the app — no compiler, no devDependencies, no raw source files. This results in a smaller image (faster to deploy, less storage/bandwidth cost) and a reduced attack surface (fewer tools an attacker could exploit if they somehow gained access to a running container).

**Interview tip:** if asked "what's a multi-stage Docker build, and why use one," the precise answer centers on separating BUILD-time needs from RUN-time needs, resulting in a smaller, more secure final image — this is genuinely one of the more advanced, senior-signaling Docker concepts to articulate clearly.

---

## 4. `.dockerignore` — a small but real detail

```
# .dockerignore -- same idea as .gitignore (Day 13), but for what
# should NOT be copied into the Docker build context at all
node_modules
npm-debug.log
.env
.git
*.test.js
```

**Why this matters:** without it, `COPY . .` would copy your LOCAL `node_modules` (built for your OS/architecture, potentially incompatible with the container's environment) into the image, bloating it and risking subtle bugs — and could accidentally bake your `.env` secrets directly into the image, a real security concern (recall Day 13's "never commit secrets" lesson, applying here too, just for Docker images instead of git).

---

## 5. docker-compose — running your full local stack with one command

### Background: why is this needed beyond just `docker run`?

A real backend app typically needs MULTIPLE services running together — your Node app, a PostgreSQL database, a Redis instance. Running each with separate `docker run` commands (with the right networking, environment variables, and startup order) is tedious and error-prone. **docker-compose** lets you define your ENTIRE local stack in one YAML file and bring it all up with a single command.

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .                  # build the image from the Dockerfile in this directory
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      # NOTE: 'db' here is the SERVICE NAME below, not 'localhost' --
      # docker-compose automatically sets up internal networking where
      # each service can reach others BY THEIR SERVICE NAME, like a
      # built-in DNS just for this stack
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
      # 'depends_on' controls START ORDER, but does NOT wait for the
      # dependency to be fully READY (e.g., Postgres accepting
      # connections) -- just that its container has STARTED. A real
      # app often needs its OWN retry/wait logic on startup, or a
      # dedicated health-check-aware wait mechanism, to handle this gap.

  db:
    image: postgres:16-alpine  # using an OFF-THE-SHELF image, not building our own
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
      # WITHOUT this volume, all database data would be LOST every time
      # the container is removed/recreated -- volumes persist data
      # OUTSIDE the container's own temporary filesystem

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data: # declares the named volume used above
```

```bash
docker-compose up           # starts ALL services defined above
docker-compose up -d        # same, but detached (runs in the background)
docker-compose down         # stops and removes all containers (but volumes persist by default)
docker-compose logs app     # view logs for just the 'app' service
docker-compose exec app sh  # open a shell INSIDE the running 'app' container -- useful for debugging
```

**Interview tip — the volumes concept specifically is worth being precise about:** if asked "what happens to my database data if I stop and restart your containers," the correct answer distinguishes between containers (ephemeral — recreating one wipes its internal filesystem) and volumes (persistent — explicitly designed to survive container recreation) — a real, practical distinction that catches people off guard the first time they lose local database data by not understanding this.

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Your Docker image is 1.2GB and deploys are slow — how would you reduce it?"** → Multi-stage builds (separating build-time tools from the runtime image), using an Alpine-based base image, and a proper `.dockerignore` to avoid copying unnecessary files.
- **"A new teammate says 'it works on my machine but not in Docker' — what would you check?"** → Whether `node_modules` is being copied from their local machine instead of installed fresh inside the container (a `.dockerignore` issue), or an environment variable difference between their local `.env` and what's configured for the container.
- **"How would you let a new developer get your entire local development environment (app + Postgres + Redis) running with one command?"** → `docker-compose up` — defining the full stack in `docker-compose.yml`, with the explanation that each service can reach the others by service name thanks to compose's built-in networking.
- **"Why might your app fail to connect to the database immediately after `docker-compose up`, even though `depends_on` is configured?"** → `depends_on` only waits for the dependency's CONTAINER to start, not for the actual service inside it (e.g., Postgres) to be fully ready to accept connections — the app needs its own retry logic, or a more sophisticated health-check-aware wait strategy.

---

## 7. Hands-on practice for today

```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

```javascript
// app.js
const express = require('express');
const app = express();

app.get('/', (req, res) => res.json({ message: 'Hello from inside a container!' }));

app.listen(3000, () => console.log('Running on port 3000'));
```

```json
// package.json
{
  "name": "docker-practice",
  "version": "1.0.0",
  "dependencies": { "express": "^4.18.0" },
  "scripts": { "start": "node app.js" }
}
```

```
# .dockerignore
node_modules
.env
.git
```

```bash
npm install   # generate package-lock.json locally first
docker build -t docker-practice .
docker run -p 3000:3000 docker-practice
curl http://localhost:3000
```

Then, as a deliberate exercise: change one line in `app.js`, rebuild (`docker build -t docker-practice .`), and watch in the build output that the `npm install` layer is **skipped/cached** (since only the source code changed, not `package.json`) — direct proof of the layer caching behavior discussed in section 2.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What's the precise difference between a container and a virtual machine?**
   A: A VM virtualizes hardware and runs a full guest OS per instance — heavy, slower to start. A container virtualizes at the OS level, sharing the host machine's kernel across all containers, packaging only the application and its specific dependencies — much lighter and faster to start.

2. **Q: Why does the order of instructions in a Dockerfile matter?**
   A: Docker caches each instruction as a layer; if an instruction's inputs haven't changed, the cached layer is reused. Copying `package.json` and running `npm install` BEFORE copying the rest of your source code means that expensive install step is skipped on rebuilds where only source code (not dependencies) changed.

3. **Q: What is a multi-stage Docker build, and why use one?**
   A: A Dockerfile with multiple `FROM` stages, where an early stage handles build-time needs (compiling TypeScript, installing devDependencies) and the FINAL stage only copies over the specific build output needed to run the app — resulting in a smaller, more secure production image without unnecessary build tools.

4. **Q: What's the difference between `EXPOSE` in a Dockerfile and the `-p` flag in `docker run`?**
   A: `EXPOSE` is documentation/metadata about which port the app inside the container listens on — it doesn't actually make the port accessible from outside. The `-p host:container` flag at runtime is what actually maps and publishes a port from the container to the host machine.

5. **Q: What happens to database data if you run `docker-compose down` without a configured volume?**
   A: It's lost — containers are ephemeral by default, and removing/recreating one wipes its internal filesystem. A named volume, mounted to the database's data directory, is required for that data to persist independently of the container's own lifecycle.

6. **Q: Does `depends_on` in docker-compose guarantee a dependency service is fully ready before your app starts?**
   A: No — it only guarantees the dependency's container has STARTED, not that the service inside it (e.g., Postgres accepting connections) is actually ready. Applications typically need their own connection retry logic, or a dedicated wait/health-check mechanism, to handle this gap reliably.

---

## Tomorrow (Day 27) Preview
We cover **CI/CD fundamentals and PM2 in production**: what a CI/CD pipeline actually automates (build, test, deploy stages), a basic GitHub Actions workflow for a Node.js app, environment strategy across dev/staging/production, and graceful shutdown handling — the operational practices that turn a working app into a properly deployed, production-grade service.
