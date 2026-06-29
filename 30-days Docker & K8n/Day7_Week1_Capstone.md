# Day 7: Week 1 Capstone — Dockerize a Full-Stack App End-to-End

**Goal for today:** Apply everything from Days 1–6 in one real, working project. You will Dockerize a complete Node.js + PostgreSQL application from scratch — writing the Dockerfile, wiring it with Docker Compose, adding healthchecks, handling secrets properly, setting up a dev environment with live reload, and running database migrations. This is not theory — every step produces something that runs.

**Format:** This day is project-driven. Concepts are introduced as you need them, not in isolation.

---

## 0. What You're Building Today

A simple but **production-pattern** REST API called **TaskAPI** — a task management backend with:

| Layer | Technology |
|---|---|
| API | Node.js + Express |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| Proxy | Nginx |
| Orchestration | Docker Compose |

**Features:**
- `GET /health` → health check endpoint
- `GET /tasks` → list all tasks (from PostgreSQL)
- `POST /tasks` → create a task
- `DELETE /tasks/:id` → delete a task

**What you'll practise from each day:**

| Day | Concept applied today |
|---|---|
| Day 1 | Understanding what each container IS and why it exists |
| Day 2 | `docker exec`, `docker logs`, `docker inspect` for debugging |
| Day 3 | Writing the Dockerfile with layer caching + multi-stage |
| Day 4 | Custom networks, DNS-based service discovery |
| Day 5 | Named volumes for PostgreSQL + bind mount for dev hot reload |
| Day 6 | Full docker-compose.yml with healthchecks + env_file |

---

## 1. Project Structure

First, create this folder structure on your machine:

```
taskapi/
├── src/
│   └── server.js          ← Express app
├── nginx/
│   └── nginx.conf         ← Nginx reverse proxy config
├── .env                   ← secrets (NOT committed to git)
├── .env.example           ← template (committed to git)
├── .dockerignore          ← exclude junk from build context
├── Dockerfile             ← production image
├── Dockerfile.dev         ← dev image (with nodemon)
├── docker-compose.yml     ← production compose
├── docker-compose.dev.yml ← dev compose override
└── package.json
```

---

## 2. Step 1 — Write the Application Code

### `package.json`
```json
{
  "name": "taskapi",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "redis": "^4.6.7",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### `src/server.js`
```javascript
require('dotenv').config();
const express = require('express');
const { Pool } = require('pg');
const { createClient } = require('redis');

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3000;

// ── PostgreSQL Connection ─────────────────────────────────────────────────────
// Notice: 'host' uses the SERVICE NAME from docker-compose.yml (Docker DNS)
const pool = new Pool({
  host: process.env.DB_HOST || 'db',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'taskdb',
  user: process.env.DB_USER || 'taskuser',
  password: process.env.DB_PASSWORD || 'secret',
});

// ── Redis Connection ───────────────────────────────────────────────────────────
const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://redis:6379'
});
redisClient.connect().catch(console.error);

// ── Database Initialisation ────────────────────────────────────────────────────
// Creates table if it doesn't exist on startup
async function initDB() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS tasks (
      id SERIAL PRIMARY KEY,
      title TEXT NOT NULL,
      done BOOLEAN DEFAULT FALSE,
      created_at TIMESTAMP DEFAULT NOW()
    )
  `);
  console.log('✅ Database initialised');
}

// ── Routes ─────────────────────────────────────────────────────────────────────

// Health check — used by Docker healthcheck + load balancers
app.get('/health', async (req, res) => {
  try {
    await pool.query('SELECT 1');              // test DB connection
    await redisClient.ping();                  // test Redis connection
    res.json({ status: 'healthy', db: 'up', redis: 'up' });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});

// List all tasks
app.get('/tasks', async (req, res) => {
  try {
    const result = await pool.query('SELECT * FROM tasks ORDER BY created_at DESC');
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Create a task
app.post('/tasks', async (req, res) => {
  const { title } = req.body;
  if (!title) return res.status(400).json({ error: 'title is required' });
  try {
    const result = await pool.query(
      'INSERT INTO tasks (title) VALUES ($1) RETURNING *',
      [title]
    );
    res.status(201).json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Delete a task
app.delete('/tasks/:id', async (req, res) => {
  try {
    await pool.query('DELETE FROM tasks WHERE id = $1', [req.params.id]);
    res.json({ message: 'Task deleted' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ── Start Server ───────────────────────────────────────────────────────────────
app.listen(PORT, async () => {
  console.log(`🚀 TaskAPI running on port ${PORT}`);
  // Retry DB init — DB may take a few seconds even after healthcheck passes
  let retries = 5;
  while (retries) {
    try {
      await initDB();
      break;
    } catch (err) {
      console.log(`⏳ DB not ready, retrying... (${retries} left)`);
      retries -= 1;
      await new Promise(res => setTimeout(res, 3000));
    }
  }
});
```

---

## 3. Step 2 — Write the Dockerfiles

### Production Dockerfile (`Dockerfile`)

This applies everything from Day 3 — layer caching trick, non-root user, exec form CMD, production deps only.

```dockerfile
# ── Stage 1: Dependency Installation ──────────────────────────────────────────
FROM node:18-alpine AS deps

WORKDIR /app

# Copy ONLY dependency files first (layer cache trick — Day 3)
COPY package*.json ./

# Install ONLY production dependencies
RUN npm install --omit=dev


# ── Stage 2: Production Image ──────────────────────────────────────────────────
FROM node:18-alpine AS production

# Install wget for healthcheck (needed by Docker healthcheck command)
RUN apk add --no-cache wget

WORKDIR /app

# Copy production node_modules from stage 1
COPY --from=deps /app/node_modules ./node_modules

# Copy application source code
COPY src/ ./src/
COPY package*.json ./

# Set ownership to non-root 'node' user (security — Day 3)
RUN chown -R node:node /app
USER node

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

# Exec form — signals go directly to node process (Day 3)
CMD ["node", "src/server.js"]
```

### Development Dockerfile (`Dockerfile.dev`)

```dockerfile
FROM node:18-alpine

# Install wget for healthcheck
RUN apk add --no-cache wget

WORKDIR /app

# Install ALL dependencies (including devDependencies like nodemon)
COPY package*.json ./
RUN npm install

# NOTE: We don't COPY source code here
# In dev, source code is bind-mounted from host (live reload)
# The COPY . . step is intentionally absent

ENV NODE_ENV=development
ENV PORT=3000

EXPOSE 3000
EXPOSE 9229

# nodemon: watches for file changes, restarts server automatically
CMD ["npx", "nodemon", "--inspect=0.0.0.0:9229", "src/server.js"]
```

**Why two Dockerfiles?**
| Dockerfile | Has source code? | Has devDependencies? | Live reload? |
|---|---|---|---|
| `Dockerfile` (prod) | ✅ Copied in | ❌ Omitted | ❌ No |
| `Dockerfile.dev` | ❌ Bind mounted | ✅ Included | ✅ nodemon |

---

## 4. Step 3 — Write the Nginx Config

### `nginx/nginx.conf`
```nginx
events {
  worker_connections 1024;
}

http {
  # Upstream block: 'api' is the Docker service name — DNS resolves it (Day 4)
  upstream api_servers {
    server api:3000;       # Docker DNS: 'api' resolves to the api container's IP
  }

  server {
    listen 80;

    # Proxy all /api/* and /health requests to the Node.js API
    location /api/ {
      proxy_pass http://api_servers/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
      proxy_pass http://api_servers/health;
    }

    # Serve a simple status page at root
    location / {
      return 200 '{"message": "TaskAPI Gateway", "status": "running"}';
      add_header Content-Type application/json;
    }
  }
}
```

---

## 5. Step 4 — Write the Environment Files

### `.env` (never commit this — add to `.gitignore`)
```env
# Database
DB_HOST=db
DB_PORT=5432
DB_NAME=taskdb
DB_USER=taskuser
DB_PASSWORD=supersecretpassword123

# Redis
REDIS_URL=redis://redis:6379

# App
PORT=3000
NODE_ENV=production
```

### `.env.example` (commit this — it's the template)
```env
# Database
DB_HOST=db
DB_PORT=5432
DB_NAME=taskdb
DB_USER=taskuser
DB_PASSWORD=              ← fill this in

# Redis
REDIS_URL=redis://redis:6379

# App
PORT=3000
NODE_ENV=production
```

### `.gitignore`
```
node_modules
.env
*.log
```

---

## 6. Step 5 — Write the `.dockerignore`

```
node_modules
.git
.env
*.log
.DS_Store
coverage
.nyc_output
docker-compose*.yml
Dockerfile*
README.md
```

---

## 7. Step 6 — Write the Production Docker Compose File

### `docker-compose.yml`

Every line here maps to something you learned in Days 4–6. Read the comments.

```yaml
version: "3.9"

services:

  # ── PostgreSQL ──────────────────────────────────────────────────────────────
  db:
    image: postgres:15-alpine
    container_name: taskapi-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: taskuser
      POSTGRES_PASSWORD: supersecretpassword123
      POSTGRES_DB: taskdb
    volumes:
      - pgdata:/var/lib/postgresql/data     # named volume: survives container deletion (Day 5)
    networks:
      - backend-net                          # isolated: only api can reach db (Day 4)
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskuser -d taskdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s                      # give postgres time to initialise

  # ── Redis ────────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: taskapi-redis
    restart: unless-stopped
    volumes:
      - redisdata:/data                      # named volume: cache persists (Day 5)
    networks:
      - backend-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # ── Node.js API ───────────────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile                 # production Dockerfile (Day 3)
    container_name: taskapi-api
    restart: unless-stopped
    env_file:
      - .env                                 # secrets from file, not hardcoded (Day 6)
    volumes:
      - uploads:/app/uploads                 # named volume: user uploads persist (Day 5)
    networks:
      - backend-net                          # talks to db and redis
      - frontend-net                         # reachable by nginx
    depends_on:
      db:
        condition: service_healthy           # wait until postgres is truly ready (Day 6)
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s

  # ── Nginx Reverse Proxy ───────────────────────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: taskapi-nginx
    restart: unless-stopped
    ports:
      - "80:80"                              # only nginx is exposed to the host (Day 4)
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro   # bind mount config, read-only (Day 5)
    networks:
      - frontend-net
    depends_on:
      - api

# ── Networks ──────────────────────────────────────────────────────────────────
networks:
  backend-net:
    driver: bridge      # db + redis + api (internal, not exposed)
  frontend-net:
    driver: bridge      # nginx + api (nginx proxies to api)

# ── Volumes ───────────────────────────────────────────────────────────────────
volumes:
  pgdata:
  redisdata:
  uploads:
```

---

## 8. Step 7 — Write the Dev Docker Compose Override

### `docker-compose.dev.yml`
```yaml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev             # use dev dockerfile (nodemon)
    container_name: taskapi-api-dev
    volumes:
      - .:/app                               # bind mount: live code reload (Day 5)
      - /app/node_modules                    # protect container's node_modules (Day 5)
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"                          # expose api directly in dev (skip nginx)
      - "9229:9229"                          # Node.js debugger port

  # In dev, expose db port for GUI tools like TablePlus / DBeaver
  db:
    ports:
      - "5432:5432"

  # In dev, expose redis port for Redis Insight / redis-cli
  redis:
    ports:
      - "6379:6379"
```

---

## 9. Step 8 — Run It

### Development (with live reload)
```bash
cd taskapi

# Start dev environment (base + dev override merged)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# Confirm all containers are healthy
docker compose ps

# API is directly accessible at:
# http://localhost:3000/health
# http://localhost:3000/tasks
```

### Production
```bash
# Start production environment
docker compose up --build -d

# API is accessible through Nginx at:
# http://localhost/health
# http://localhost/api/tasks
```

---

## 10. Step 9 — Verify Everything Works (Test Your Stack)

Run these in order. Each tests a different layer of the stack.

```bash
# ── 1. Check all containers are running and healthy ───────────────────────────
docker compose ps

# Expected output:
# NAME             IMAGE              STATUS
# taskapi-db       postgres:15-alpine running (healthy)
# taskapi-redis    redis:7-alpine     running (healthy)
# taskapi-api      taskapi-api        running (healthy)
# taskapi-nginx    nginx:alpine       running


# ── 2. Test health endpoint ───────────────────────────────────────────────────
curl http://localhost:3000/health
# Expected: {"status":"healthy","db":"up","redis":"up"}


# ── 3. Create some tasks ──────────────────────────────────────────────────────
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Docker Compose"}'

curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Dockerize PulseBloom"}'


# ── 4. List tasks ─────────────────────────────────────────────────────────────
curl http://localhost:3000/tasks
# Expected: array of tasks from PostgreSQL


# ── 5. Test DNS resolution (Day 4 concept) ────────────────────────────────────
# From inside the api container, ping db by name — Docker DNS resolves it
docker exec -it taskapi-api ping db
docker exec -it taskapi-api ping redis


# ── 6. Test data persistence (Day 5 concept) ──────────────────────────────────
# Restart the db container
docker compose restart db

# Wait for it to become healthy
docker compose ps

# Tasks are still there (volume survives restart)
curl http://localhost:3000/tasks


# ── 7. Test volume persistence across container deletion ──────────────────────
docker compose down          # stop + remove containers (NOT volumes)
docker compose up -d         # bring back up
curl http://localhost:3000/tasks   # ✅ data still there


# ── 8. View logs from all services ────────────────────────────────────────────
docker compose logs -f
docker compose logs -f api    # only api logs


# ── 9. Enter the database directly ───────────────────────────────────────────
docker compose exec db psql -U taskuser -d taskdb
# Inside psql:
# \dt               → list tables
# SELECT * FROM tasks;
# \q                → quit


# ── 10. Test live reload (dev only) ───────────────────────────────────────────
# Edit src/server.js — add a new route or console.log
# Watch the api container logs — nodemon restarts automatically
docker compose -f docker-compose.yml -f docker-compose.dev.yml logs -f api
```

---

## 11. Step 10 — Debugging Checklist

When something breaks (and it will — that's normal), use this checklist:

```bash
# Container not starting?
docker compose logs api            # read the error message

# Container starting but app crashing?
docker compose logs api --tail 50  # last 50 lines
docker compose ps                  # check STATUS (exited? restarting?)

# Can't connect to database?
docker compose exec api ping db    # DNS resolution working?
docker compose exec db pg_isready -U taskuser  # is postgres ready?
docker inspect taskapi-db | grep '"Networks"' -A 20  # is db on the right network?

# Port not accessible from browser?
docker compose ps                  # is port mapping listed?
curl http://localhost:3000/health  # curl directly to bypass any proxy issues

# Volume data not persisting?
docker volume ls                   # is the volume created?
docker volume inspect taskapi_pgdata  # check mountpoint

# Image not reflecting Dockerfile changes?
docker compose up --build          # force rebuild

# Nuclear option: full reset (⚠️ deletes all data)
docker compose down -v --rmi all
docker compose up --build -d
```

---

## 12. Week 1 — Complete Concept Review

Use this as your teaching outline — you should be able to walk through every row from memory:

| Concept | What it is | Key command/file | Real use in today's project |
|---|---|---|---|
| Container | Isolated process using host OS kernel | `docker run` | Each service (api, db, redis, nginx) is a container |
| Image | Read-only layered blueprint | `docker build` | `Dockerfile` builds the api image |
| Layer caching | Reuse unchanged build steps | Instruction order in Dockerfile | `package*.json` copied before `COPY . .` |
| Multi-stage build | Separate build + runtime stages | `FROM ... AS stage` | `deps` stage → `production` stage |
| `.dockerignore` | Exclude files from build context | `.dockerignore` | Excludes `node_modules`, `.env`, logs |
| Bridge network | Virtual network for container communication | `docker network create` | `backend-net` and `frontend-net` |
| Docker DNS | Container name resolution on custom networks | Container name as hostname | `host: 'db'` in server.js, `server api:3000` in nginx |
| Port mapping | Expose container port to host | `-p host:container` | Only nginx exposed on `:80` |
| Named volume | Persistent Docker-managed storage | `-v name:/path` | `pgdata`, `redisdata`, `uploads` |
| Bind mount | Host path → container path | `-v $(pwd):/app` | Dev: live code reload |
| Anonymous volume | Shield path from bind mount | `-v /path` | Protects `/app/node_modules` |
| Docker Compose | Multi-container stack management | `docker compose up` | `docker-compose.yml` runs entire stack |
| `depends_on` + healthcheck | Proper startup ordering | `condition: service_healthy` | API waits for DB to be truly ready |
| `env_file` | Load secrets from file | `env_file: .env` | DB credentials not hardcoded in YAML |
| Dev vs prod Compose | Environment-specific configs | `-f base.yml -f override.yml` | `docker-compose.dev.yml` adds bind mounts + nodemon |

---

## 13. Week 2 Preview

You've now mastered the Docker fundamentals. In Week 2 we begin the transition to Kubernetes:

- **Day 8** — Docker best practices: multi-stage builds, image size optimisation, security scanning
- **Day 9** — Registries: pushing your image to Docker Hub and AWS ECR (directly relevant to your ECS Fargate deploys)
- **Day 10** — Why orchestration? The problems Docker Compose can't solve at scale — this is where Kubernetes comes in
- **Day 11** — Kubernetes architecture: control plane, nodes, and how it differs from Docker

---

## 14. Quiz — Week 1 Final Test

Test yourself on every day of the week:

1. What is the fundamental difference between a container and a virtual machine?
2. What happens to data written inside a container when you `docker rm` it?
3. In a Dockerfile, why do we `COPY package*.json ./` before `COPY . .`?
4. What does the `AS builder` part of `FROM node:18-alpine AS builder` do?
5. Why must you use a **custom** bridge network (not the default) for container-to-container DNS?
6. In `server.js`, why can we use `host: 'db'` instead of an IP address?
7. What's the difference between a named volume and a bind mount? When do you use each?
8. What problem does the anonymous volume `-v /app/node_modules` solve?
9. What is the critical limitation of `depends_on`, and how do healthchecks fix it?
10. What is the difference between `docker compose down` and `docker compose down -v`?
11. Why do we have two separate Docker Compose files (`docker-compose.yml` and `docker-compose.dev.yml`)?
12. In the Nginx config, why can we write `server api:3000` and have it work?

<details>
<summary>Answers</summary>

1. VMs carry a full guest OS (GBs, boots in minutes). Containers share the host OS kernel and only isolate the application layer (MBs, starts in seconds).
2. Data written to the container's writable layer is permanently deleted with it. Data in named volumes or bind mounts is NOT affected.
3. `package*.json` changes rarely (only when you add/remove packages). Copying it first lets Docker cache the `npm install` layer and skip re-running it on every code change — massive build speed improvement.
4. It names that build stage `builder` so that a later stage can reference it with `COPY --from=builder` to copy only the build artifacts, leaving behind all build tools and source.
5. The default bridge network only allows container communication by IP address. Custom bridge networks get Docker's built-in DNS server, enabling name-based discovery.
6. Docker's built-in DNS on custom bridge networks resolves the container name `db` to its current IP automatically. This is more reliable than IPs, which can change on restarts.
7. Named volume: Docker-managed storage in Docker's internal area — use for production data persistence (databases, uploads). Bind mount: maps a specific host path into the container — use for local dev to sync code changes.
8. When you bind mount `$(pwd):/app`, it overlays your host folder onto `/app`, wiping out the container's installed `node_modules`. The anonymous volume on `/app/node_modules` "shields" that directory from the overlay, keeping the container's own modules intact.
9. `depends_on` only waits for the container to start, not for the service inside to be ready. PostgreSQL can take several seconds to initialize after the container starts. Adding a `healthcheck` and using `condition: service_healthy` makes Compose wait until the actual service passes its health test.
10. `docker compose down` removes containers and networks but keeps volumes (data safe). `docker compose down -v` also removes all named volumes, permanently deleting all persisted data.
11. Separation of concerns: the base file has shared production config. The dev override adds development-only concerns (live code bind mount, nodemon, exposed debug ports, exposed DB ports for GUI tools) without polluting the production file.
12. `api` is the service name in the Compose file. Docker creates a `frontend-net` custom bridge network and registers DNS for all services on it. Nginx (on `frontend-net`) resolves `api` to the api container's IP automatically.

</details>

---

## 15. Summary

Today you built a complete, real project that applies every concept from the week:
- A multi-stage production Dockerfile with security best practices
- A dev Dockerfile with nodemon and debugger support
- A full `docker-compose.yml` with healthchecks, env_file, named volumes, and multi-network isolation
- A dev override file with live code reload via bind mount
- Nginx as a reverse proxy using Docker DNS to reach the API
- A systematic debugging checklist for when things break
- A full week 1 concept review table you can use for teaching

**You now know Docker. Week 2 starts the path to Kubernetes.**
