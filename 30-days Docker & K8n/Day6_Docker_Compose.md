# Day 6: Docker Compose — Managing Multi-Container Apps Like a Pro

**Goal for today:** Understand what Docker Compose is, why it exists, every important field in a `docker-compose.yml` file, and how to use it to define, run, and manage a complete multi-container application with a single command. By the end, you should be able to replace every `docker run` command you've written this week with a clean, version-controlled Compose file.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

This week you learned the four Docker primitives:
- **Day 3** → Images (Dockerfile, layers, caching)
- **Day 4** → Networks (bridge, DNS, port mapping)
- **Day 5** → Volumes (named volumes, bind mounts, tmpfs)
- **Days 1–2** → Containers (the running instances of images)

**Today's connecting thought:**
> By Day 5, running a full stack (API + PostgreSQL + Redis) required 3 long `docker run` commands, 2 `docker network create` commands, and 3 `docker volume create` commands — typed manually, every single time. One typo breaks everything. No way to version-control it.
>
> Docker Compose solves this: you write the entire stack once in a YAML file, commit it to Git, and start the whole thing with `docker compose up`. That's it.

---

## 1. What is Docker Compose?

### WHAT
Docker Compose is a tool for defining and running **multi-container Docker applications** using a single YAML configuration file called `docker-compose.yml`. It lets you declare all your containers (called **services**), their networks, volumes, environment variables, dependencies, and build instructions in one place.

### WHY
Without Compose, managing even a simple 3-container app (API + DB + Cache) means:
- Remembering 10+ flags across multiple `docker run` commands
- Manually creating networks and volumes before each run
- No easy way to bring everything up or tear everything down together
- No version control for your infrastructure config

With Compose:
- **One file** describes the entire stack
- **One command** (`docker compose up`) starts everything
- **One command** (`docker compose down`) stops and cleans everything
- The file lives in Git alongside your code — infrastructure as code

### HOW (the big picture)
```
docker-compose.yml
        │
        ▼
docker compose up
        │
        ├── Creates networks defined in the file
        ├── Creates volumes defined in the file
        ├── Builds images (if build: is specified)
        ├── Pulls images (if image: is specified)
        └── Starts all containers in the right order

docker compose down
        │
        ├── Stops all containers
        ├── Removes all containers
        ├── Removes networks
        └── (Volumes kept by default — data safe!)
```

**Teaching line:**
> "If a Dockerfile is a recipe for one dish, `docker-compose.yml` is the full restaurant menu — it describes every dish, how they're made, and how they're served together."

---

## 2. docker-compose.yml — File Structure

Every Compose file has this top-level structure:

```yaml
version: "3.9"          # Compose file format version (optional in newer Docker)

services:               # ← Define your containers here (required)
  service-name-1:
    ...
  service-name-2:
    ...

networks:               # ← Define custom networks (optional)
  network-name:
    ...

volumes:                # ← Define named volumes (optional)
  volume-name:
    ...
```

Each **service** = one container configuration.

---

## 3. Every Important Field in a Service — WHAT, WHY, HOW

---

### 3.1 `image` — Which image to use

```yaml
services:
  db:
    image: postgres:15-alpine
```
Pulls and runs the specified image from a registry (Docker Hub by default).

---

### 3.2 `build` — Build an image from a Dockerfile

```yaml
services:
  api:
    build: .                          # Dockerfile in current directory
    # OR more explicitly:
    build:
      context: .                      # build context folder
      dockerfile: Dockerfile.dev      # specific Dockerfile name
```

**WHAT:** Tells Compose to build an image using a Dockerfile instead of pulling one.
**WHY:** Your own app doesn't exist on Docker Hub — you need to build it locally.
**HOW:** When you run `docker compose up`, Compose builds the image first, then starts the container from it. Use `docker compose build` to build without starting.

---

### 3.3 `container_name` — Give the container a fixed name

```yaml
services:
  api:
    container_name: queuecare-api
```

By default Compose names containers `<project>_<service>_1`. A fixed name is easier to reference with `docker exec`, `docker logs`, etc.

---

### 3.4 `ports` — Port mapping

```yaml
services:
  api:
    ports:
      - "3000:3000"        # host:container
      - "9229:9229"        # debugger port
```
Same as `-p 3000:3000` in `docker run`.

---

### 3.5 `environment` — Set environment variables

```yaml
services:
  db:
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret123
      POSTGRES_DB: appdb
```

Or using a list format:
```yaml
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=secret123
```

---

### 3.6 `env_file` — Load env vars from a file

```yaml
services:
  api:
    env_file:
      - .env              # loads all KEY=VALUE lines from .env file
```

**WHY this is better than hardcoding:** Secrets don't end up in your Compose file (which is in Git). `.env` is in `.gitignore`. This is the production-safe pattern.

---

### 3.7 `volumes` — Mount volumes or bind mounts

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data     # named volume

  api:
    volumes:
      - .:/app                              # bind mount: current dir → /app
      - /app/node_modules                   # anonymous volume: protect node_modules
      - ./logs:/app/logs                    # bind mount: specific subfolder
```

---

### 3.8 `networks` — Connect to specific networks

```yaml
services:
  api:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net
```

**Important:** If you don't specify `networks` in a service, Compose automatically creates a default network for the whole project and attaches all services to it. So by default, all services in one Compose file can talk to each other by service name — no explicit network config needed for simple setups.

---

### 3.9 `depends_on` — Control startup order

```yaml
services:
  api:
    depends_on:
      - db
      - redis
```

**WHAT:** Tells Compose to start `db` and `redis` before starting `api`.
**WHY:** Your API will crash on startup if it tries to connect to a database that hasn't started yet.
**Critical limitation:** `depends_on` only waits for the container to **start**, not for the service inside to be **ready**. PostgreSQL takes a few seconds to initialize even after the container starts. This is the #1 `depends_on` gotcha.

**The proper fix — use healthchecks (Section 4):**
```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy     # wait until db passes its healthcheck
```

---

### 3.10 `restart` — Restart policy

```yaml
services:
  api:
    restart: unless-stopped
```

| Policy | Behaviour |
|---|---|
| `no` | Never restart (default) |
| `always` | Always restart when stopped for any reason |
| `on-failure` | Restart only if the container exits with an error code |
| `unless-stopped` | Always restart unless you explicitly `docker compose stop` it |

Use `unless-stopped` for production services.

---

### 3.11 `command` — Override the default CMD

```yaml
services:
  api:
    command: npx nodemon server.js    # dev: use nodemon for hot reload
```

Overrides whatever `CMD` is in the Dockerfile — useful for dev vs prod differences.

---

### 3.12 `working_dir` — Set working directory

```yaml
services:
  api:
    working_dir: /app
```

---

### 3.13 `healthcheck` — Define a health test

```yaml
services:
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s       # check every 10 seconds
      timeout: 5s         # fail the check if it takes more than 5s
      retries: 5          # mark unhealthy after 5 consecutive failures
      start_period: 30s   # wait 30s before starting checks (give app time to boot)
```

**WHAT:** Defines a command Docker runs periodically to check if the service is healthy.
**WHY:** Enables `depends_on: condition: service_healthy` — so the API truly waits until PostgreSQL is ready, not just started.

---

## 4. Full Real-World Compose File — QueueCare / PulseBloom Pattern

This is the Compose equivalent of everything you did manually across Days 4 and 5. Read every line — each maps to something you've already learned.

```yaml
# docker-compose.yml

version: "3.9"

# ─── Services ──────────────────────────────────────────────────────────────────
services:

  # ── PostgreSQL Database ──────────────────────────────────────────────────────
  db:
    image: postgres:15-alpine
    container_name: app-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret123
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data     # named volume: data persists
    networks:
      - backend-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ── Redis Cache ───────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: app-redis
    restart: unless-stopped
    volumes:
      - redisdata:/data                     # named volume: cache persists
    networks:
      - backend-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # ── Node.js / Express API ─────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    env_file:
      - .env                                # secrets loaded from .env (not committed)
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://appuser:secret123@db:5432/appdb
      REDIS_URL: redis://redis:6379
    volumes:
      - uploads:/app/uploads                # named volume: file uploads persist
    networks:
      - backend-net
      - frontend-net
    depends_on:
      db:
        condition: service_healthy          # truly wait until postgres is ready
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 15s

  # ── Nginx Reverse Proxy (optional frontend gateway) ───────────────────────────
  nginx:
    image: nginx:alpine
    container_name: app-nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro   # bind mount config, read-only
    networks:
      - frontend-net
    depends_on:
      - api

# ─── Networks ──────────────────────────────────────────────────────────────────
networks:
  backend-net:
    driver: bridge      # api, db, redis — internal communication only
  frontend-net:
    driver: bridge      # nginx, api — internet-facing services

# ─── Volumes ───────────────────────────────────────────────────────────────────
volumes:
  pgdata:               # postgres data
  redisdata:            # redis data
  uploads:              # user uploaded files
```

**Start it all:**
```bash
docker compose up -d           # start everything in background
docker compose logs -f         # tail all logs
docker compose logs -f api     # tail only api logs
```

**Tear it all down:**
```bash
docker compose down            # stop + remove containers + networks (volumes kept)
docker compose down -v         # also remove volumes (⚠️ deletes all data)
```

---

## 5. Development vs Production Compose Files

Real projects use **multiple Compose files** — a base, a dev override, and a production override. This is the professional pattern.

### Base file: `docker-compose.yml` (shared config)
```yaml
services:
  api:
    build: .
    environment:
      NODE_ENV: production
  db:
    image: postgres:15-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Dev override: `docker-compose.dev.yml`
```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev       # dev Dockerfile with nodemon
    command: npx nodemon server.js
    volumes:
      - .:/app                          # live code bind mount
      - /app/node_modules
    environment:
      NODE_ENV: development
    ports:
      - "9229:9229"                     # Node.js debugger port
```

### Run with the override merged in:
```bash
# Development (base + dev override merged)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production (base only)
docker compose up
```

**Teaching line:**
> "The base file is the foundation of your house. Each override file adds a different kind of furniture for different occasions — developer tools for dev, performance configs for prod. You can mix and match."

---

## 6. Essential Docker Compose CLI Commands

```bash
# ── Lifecycle ──────────────────────────────────────────────────────────────────
docker compose up                  # start all services (foreground)
docker compose up -d               # start all services (background / detached)
docker compose up --build          # rebuild images before starting
docker compose up api              # start only the 'api' service (and its deps)

docker compose down                # stop + remove containers + networks
docker compose down -v             # also remove volumes (⚠️ data loss)
docker compose down --rmi all      # also remove built images

docker compose start               # start existing (already created) containers
docker compose stop                # stop containers (keep them, don't remove)
docker compose restart api         # restart a specific service

# ── Build ──────────────────────────────────────────────────────────────────────
docker compose build               # build all images
docker compose build api           # build only 'api' image
docker compose build --no-cache    # force rebuild without cache

# ── Monitoring ─────────────────────────────────────────────────────────────────
docker compose ps                  # list all service containers
docker compose logs                # show logs from all services
docker compose logs -f             # follow all logs live
docker compose logs -f api db      # follow specific services
docker compose top                 # show running processes inside containers

# ── Running Commands ───────────────────────────────────────────────────────────
docker compose exec api bash              # open shell in running api container
docker compose exec db psql -U postgres   # open psql in running db container
docker compose run --rm api node migrate.js  # one-off command (new container, auto-removed)

# ── Scaling ────────────────────────────────────────────────────────────────────
docker compose up -d --scale api=3   # run 3 instances of the api service

# ── Config Validation ──────────────────────────────────────────────────────────
docker compose config              # validate and view the merged Compose config
```

---

## 7. `docker compose exec` vs `docker compose run`

This distinction trips up beginners — teach it explicitly.

| Command | What it does |
|---|---|
| `docker compose exec api bash` | Open a shell in the **already running** `api` container |
| `docker compose run --rm api bash` | Start a **brand new** container from the `api` service config and run bash in it |

**When to use `run`:**
- Running database migrations (`docker compose run --rm api npm run migrate`)
- One-off scripts that need your service's environment but shouldn't be in the main container
- Running tests in an isolated container

---

## 8. Common Compose Mistakes to Teach/Avoid

| Mistake | Problem | Fix |
|---|---|---|
| Hardcoding secrets in Compose file | Secrets committed to Git | Use `env_file: .env` |
| Using `depends_on` without healthcheck | API starts before DB is actually ready | Add `condition: service_healthy` |
| `docker compose down -v` carelessly | Destroys all database data | Only use `-v` when you intentionally want to reset data |
| Not specifying networks | All services on one flat network | Define frontend-net/backend-net for proper isolation |
| One Compose file for both dev and prod | Dev tools in prod image (nodemon, etc.) | Use base + override pattern |
| Forgetting `--build` flag after Dockerfile change | Running stale old image | `docker compose up --build` after any Dockerfile edit |

---

## 9. Hands-On Practice Tasks

1. Take the long `docker run` commands you wrote across Days 4 and 5 and rewrite them as a `docker-compose.yml`
2. Add healthchecks to the database service and use `condition: service_healthy` in `depends_on`
3. Create a `.env` file with your secrets and use `env_file` in the Compose service
4. Run `docker compose up -d` and then `docker compose ps` to confirm all services are running
5. Run `docker compose logs -f` and watch all service logs simultaneously
6. Run `docker compose exec db psql -U postgres` and create a table
7. Run `docker compose down` (without `-v`) and bring it back up — confirm the DB table is still there
8. Run `docker compose down -v` and bring it back up — confirm data is gone
9. Try `docker compose up --scale api=2` and run `docker compose ps` to see 2 api instances

---

## 10. Quiz — Test Yourself / Test Others

1. What problem does Docker Compose solve over plain `docker run` commands?
2. What is a "service" in Docker Compose terminology?
3. What does `depends_on` do, and what is its critical limitation?
4. How does `condition: service_healthy` improve on plain `depends_on`?
5. What is the difference between `docker compose down` and `docker compose down -v`?
6. What is the difference between `docker compose exec` and `docker compose run`?
7. Why should you use `env_file: .env` instead of hardcoding secrets in the Compose file?
8. If you update your `Dockerfile`, what flag must you add to `docker compose up` to pick up the change?

<details>
<summary>Answers</summary>

1. Compose lets you define an entire multi-container stack (containers, networks, volumes) in a single YAML file and manage it with simple commands, replacing many complex `docker run` commands with `docker compose up/down`.
2. A service is one container configuration — it maps to a single container type in your application (e.g., `api`, `db`, `redis`). Compose can run multiple instances (replicas) of one service.
3. `depends_on` ensures Compose starts listed services first before starting the dependent service. The limitation: it only waits for the container to start, not for the actual application inside (e.g., PostgreSQL) to be ready to accept connections.
4. `condition: service_healthy` makes the dependent service wait until the dependency passes its defined `healthcheck` test — meaning the application inside is truly ready, not just the container is running.
5. `docker compose down` stops and removes containers and networks but keeps volumes (data safe). `docker compose down -v` also removes all named volumes — permanently deleting all persisted data.
6. `exec` runs a command inside an **already running** container. `run` starts a **brand new** container from a service's config to run a one-off command.
7. Secrets in the Compose file get committed to Git — a serious security risk. `env_file: .env` loads secrets from a local file that lives in `.gitignore` and never gets committed.
8. `docker compose up --build` — without this flag, Compose reuses the previously built image and ignores Dockerfile changes.

</details>

---

## 11. Summary

After today you can explain and implement:
- What Docker Compose is and why it's essential for multi-container apps
- Every major field: `image`, `build`, `ports`, `environment`, `env_file`, `volumes`, `networks`, `depends_on`, `restart`, `healthcheck`, `command`
- A full production-quality Compose file for a Node.js + PostgreSQL + Redis + Nginx stack
- Dev vs production Compose file splitting pattern
- The full CLI: `up`, `down`, `build`, `logs`, `exec`, `run`, `ps`, `scale`, `config`
- The `exec` vs `run` distinction
- Common pitfalls: secrets in YAML, `depends_on` without healthcheck, `-v` flag risks

**Coming up on Day 7:** Week 1 capstone — you'll Dockerize a complete full-stack Node.js + PostgreSQL app end-to-end: write the Dockerfile, wire it with Compose, add a healthcheck, handle environment configs, and have a working local dev environment with live reload — everything from Days 1–6 applied in one real project.
