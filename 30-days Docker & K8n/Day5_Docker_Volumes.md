# Day 5: Docker Volumes & Data Persistence

**Goal for today:** Understand why containers lose data by default, what Docker volumes are, the three ways to mount storage (named volumes, bind mounts, tmpfs), when to use each, and how to make stateful apps like PostgreSQL survive container restarts and deletions. By the end, you should be able to set up persistent storage for any container and teach the concept from scratch.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 1: Containers = lightweight isolated processes sharing host OS kernel
- Day 2: Core CLI — `run`, `exec`, `logs`, `inspect`, `ps`
- Day 3: Dockerfile — building custom images, layer caching, multi-stage builds
- Day 4: Networking — custom bridge networks, Docker DNS, port mapping

**Today's connecting thought:**
> Every container we've run so far is **ephemeral** — when it's deleted, everything inside it disappears. That's fine for a stateless API, but catastrophic for a database. Today we solve the fundamental question: *"How do I make sure my data survives when a container dies?"*

---

## 1. The Problem — Containers Are Ephemeral By Design

### WHAT
Remember from Day 2 — a container has a thin **writable layer** on top of read-only image layers. Any data written inside a running container (files created, database rows inserted, logs written) goes into that writable layer.

**When you delete the container, the writable layer is deleted with it. All data is gone.**

### WHY this is actually a feature, not a bug
Ephemeral containers are a design choice that brings powerful advantages:
- **Reproducibility** — every container starts from a clean, known state
- **Stateless scaling** — you can spin up 10 identical containers because none carry unique state
- **Easy upgrades** — delete old container, start new one with new image version, no migration headache

**But stateful apps (databases, file uploads, user-generated content) NEED data to outlive containers.**

### Demonstrating the problem
```bash
# Start a container and create a file inside it
docker run -it --name demo ubuntu bash
# Inside the container:
echo "important data" > /data/myfile.txt
exit

# Delete the container
docker rm demo

# Start a brand new container from the same image
docker run -it --name demo2 ubuntu bash
# Inside new container:
cat /data/myfile.txt     # ❌ ERROR — file doesn't exist. Data is gone.
```

**This is the problem volumes solve.**

---

## 2. The Solution — Docker Volumes

### WHAT
A **volume** is a storage mechanism that lives **outside the container's writable layer** — it exists independently on the host machine (managed by Docker) and can be mounted into one or more containers. Data written to a volume persists even after the container is deleted.

### WHY
Volumes decouple the **data lifecycle** from the **container lifecycle**:
- Container can be deleted, upgraded, replaced → volume (and its data) remains
- Multiple containers can share the same volume (useful for shared file access)
- Docker manages volumes in a dedicated area (`/var/lib/docker/volumes/` on Linux) — no manual filesystem management needed
- Volumes work on all platforms (Linux, Mac, Windows) uniformly

### Three types of mounts (the full picture)

```
┌────────────────────────────────────────────────────────────┐
│                    Host Machine                              │
│                                                             │
│  ┌──────────────────────────────────────────┐              │
│  │           Container                        │              │
│  │                                            │              │
│  │   /app/data  ──────────────────────────────┼──► Named Volume (Docker-managed)
│  │   /app/code  ──────────────────────────────┼──► Bind Mount (host directory)
│  │   /tmp/cache ──────────────────────────────┼──► tmpfs Mount (RAM, never on disk)
│  │                                            │              │
│  └──────────────────────────────────────────┘              │
└────────────────────────────────────────────────────────────┘
```

---

## 3. Type 1 — Named Volumes (Recommended for Production)

### WHAT
A **named volume** is a storage area completely managed by Docker. You give it a name, Docker creates and manages it in its own internal storage area (`/var/lib/docker/volumes/<name>/_data` on Linux). The container mounts it at a path of your choosing.

### WHY
Named volumes are the **recommended way** to persist data for production:
- Docker handles creation, deletion, and lifecycle
- Portable — works the same on any machine/OS
- Can be backed up, migrated, restored using Docker commands
- Multiple containers can share one volume simultaneously
- Not tied to any specific host directory path (unlike bind mounts)

### HOW

**Create a named volume:**
```bash
docker volume create my-data
```

**Mount it into a container:**
```bash
docker run -d \
  --name postgres \
  -v my-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine
```

**Syntax:** `-v <volume-name>:<path-inside-container>`

**What happens:**
- Docker mounts `my-data` volume at `/var/lib/postgresql/data` inside the container
- PostgreSQL writes all its database files to `/var/lib/postgresql/data`
- Those files land in the `my-data` volume on the host
- Delete the container → volume and all data survives
- Run a new postgres container with the same `-v my-data:/var/lib/postgresql/data` → all data is back

**Prove data persists:**
```bash
# Start postgres with named volume
docker run -d --name pg1 \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# Create a table and insert data
docker exec -it pg1 psql -U postgres -c "CREATE TABLE users (id SERIAL, name TEXT);"
docker exec -it pg1 psql -U postgres -c "INSERT INTO users (name) VALUES ('Ashish');"

# Delete the container
docker rm -f pg1

# Start a NEW container with the SAME volume
docker run -d --name pg2 \
  -v pgdata:/var/lib/postgresql/data \    # ← same volume name!
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# Data is still there!
docker exec -it pg2 psql -U postgres -c "SELECT * FROM users;"
# Returns: Ashish ✅
```

**Teaching line:**
> "A named volume is like a hard drive you can unplug from one computer and plug into another. The computer (container) can be replaced, but the hard drive (volume) with all your data stays intact."

---

## 4. Volume CLI Commands — Full Reference

```bash
# Create a named volume
docker volume create my-volume

# List all volumes
docker volume ls

# Inspect a volume (see mountpoint, creation date, driver)
docker volume inspect my-volume

# Remove a specific volume (only if no container is using it)
docker volume rm my-volume

# Remove ALL unused volumes (not mounted to any container)
docker volume prune

# See which volumes a container is using
docker inspect my-container | grep -A 10 '"Mounts"'
```

**Sample output of `docker volume inspect`:**
```json
[
  {
    "Name": "pgdata",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
    "CreatedAt": "2025-01-01T10:00:00Z",
    "Labels": {},
    "Scope": "local"
  }
]
```

---

## 5. Type 2 — Bind Mounts (Best for Development)

### WHAT
A **bind mount** maps a **specific directory or file on the host machine** directly into the container. Unlike named volumes, you control exactly which host path is mounted — there's no Docker-managed storage area.

### WHY
Bind mounts are **essential for local development**:
- Mount your source code directory into the container → edit code on your host with your IDE → changes instantly reflect inside the container → no need to rebuild the image on every change
- Mount config files from the host into the container at runtime
- Mount log directories to easily access container logs from the host

### HOW

**Syntax:** `-v <absolute-host-path>:<container-path>`

```bash
# Mount current directory into container (live code editing)
docker run -d \
  --name dev-api \
  -v $(pwd):/app \           # host: current folder → container: /app
  -p 3000:3000 \
  -w /app \
  node:18-alpine \
  node server.js
```

Now if you edit `server.js` on your host with VS Code, the container immediately sees the updated file (though you may still need to restart the Node process — this is where `nodemon` helps).

**With nodemon for hot reload:**
```bash
docker run -d \
  --name dev-api \
  -v $(pwd):/app \
  -v /app/node_modules \     # ← exclude node_modules from the bind mount (see Section 7)
  -p 3000:3000 \
  -w /app \
  node:18-alpine \
  npx nodemon server.js      # auto-restarts on file changes
```

**Mount a specific config file:**
```bash
docker run -d \
  --name nginx \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf \   # override config file only
  -p 80:80 \
  nginx
```

**Teaching line:**
> "A bind mount is like giving the container a window directly into your host's filesystem. What the container sees through that window and what you see from your IDE are the same files in real time."

---

## 6. Named Volume vs Bind Mount — The Comparison

| Aspect | Named Volume | Bind Mount |
|---|---|---|
| Managed by | Docker | You (host path) |
| Location on host | Docker's internal area | Anywhere you specify |
| Created automatically? | ✅ Yes | ❌ No (host path must exist or Docker creates dir) |
| Best for | Production data persistence | Local development, config files |
| Portability | ✅ High (no host path hardcoded) | ❌ Low (path must match on each machine) |
| Performance (on Mac/Win) | ✅ Fast | ⚠️ Slower (filesystem translation overhead) |
| Sharing between containers | ✅ Yes | ✅ Yes |
| Backup/restore with Docker | ✅ Easy | Manual |

**Rule of thumb to teach:**
> - **Named volume** → anything that needs to persist in production (database files, uploaded files, certs)
> - **Bind mount** → local development (source code, config overrides)

---

## 7. Type 3 — tmpfs Mounts (In-Memory, Never on Disk)

### WHAT
A **tmpfs mount** stores data in the host's **RAM** — it is never written to disk. The data exists only as long as the container is running and is lost the moment the container stops.

### WHY
Use tmpfs for:
- **Sensitive data** — secrets, tokens, session data that should never touch disk at all
- **High-speed temporary storage** — caches, scratch space that benefits from RAM speed
- **Security compliance** — some environments require secrets to never be persisted to disk

### HOW
```bash
docker run -d \
  --name secure-app \
  --tmpfs /app/secrets \          # this directory lives only in RAM
  --tmpfs /tmp:size=100m \        # with size limit
  my-secure-app
```

---

## 8. The `node_modules` Problem in Bind Mounts

This is a classic gotcha that trips up every Node.js developer using Docker for local dev. Teach this explicitly.

### The Problem
```bash
# You bind mount your project folder into the container
docker run -v $(pwd):/app node:18-alpine node server.js
```
Your project folder on the host might have no `node_modules` (or different ones compiled for Mac/Windows). The bind mount *replaces* the container's `/app` with your host folder — now `node_modules` inside the container is gone or wrong, and `node server.js` fails.

### The Fix — Anonymous Volume Trick
```bash
docker run -d \
  -v $(pwd):/app \                  # bind mount: sync all code from host
  -v /app/node_modules \            # ← anonymous volume: keeps container's node_modules untouched
  -w /app \
  -p 3000:3000 \
  node:18-alpine \
  node server.js
```

`-v /app/node_modules` (no host path, just a container path) creates an **anonymous volume** that Docker manages. It "shields" the `node_modules` directory from being overwritten by the bind mount, keeping the container's own npm-installed modules intact.

**Teaching line:**
> "Think of it as: the bind mount paints the entire `/app` wall with your host files — but the anonymous volume stamps 'DO NOT PAINT' on the `node_modules` area."

---

## 9. Real-World Example — Full Persistent Stack (QueueCare / PulseBloom Pattern)

This is how you'd run your actual stack with full data persistence, before Docker Compose (Day 6):

```bash
# Step 1: Create network
docker network create app-net

# Step 2: Create named volumes
docker volume create postgres-data
docker volume create redis-data
docker volume create app-uploads

# Step 3: Start PostgreSQL with persistent volume
docker run -d \
  --name postgres \
  --network app-net \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=appdb \
  postgres:15-alpine

# Step 4: Start Redis with persistent volume
docker run -d \
  --name redis \
  --network app-net \
  -v redis-data:/data \
  redis:7-alpine

# Step 5: Start your API
# - Bind mount source code for dev (remove in production, use COPY in Dockerfile)
# - Named volume for user uploads
docker run -d \
  --name api \
  --network app-net \
  -v $(pwd):/app \               # dev: live code editing
  -v /app/node_modules \         # protect container's node_modules
  -v app-uploads:/app/uploads \  # persistent file uploads
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://appuser:secret123@postgres:5432/appdb \
  -e REDIS_URL=redis://redis:6379 \
  node:18-alpine \
  npx nodemon server.js
```

**Now you have:**
- ✅ Code changes reflected live (bind mount)
- ✅ Database data persists across restarts (named volume)
- ✅ Redis data persists across restarts (named volume)
- ✅ Uploaded files persist across restarts (named volume)
- ✅ Containers can communicate (custom network)

---

## 10. Backing Up and Restoring Volumes

### WHAT
Volumes need backups just like any database. Since Docker manages the storage location, you use a temporary container to access the volume and create a tar archive.

### HOW

**Backup a volume:**
```bash
# Run a temporary container that:
# 1. Mounts the volume you want to backup
# 2. Mounts your current host directory
# 3. Creates a tar archive of the volume into your host directory
docker run --rm \
  -v pgdata:/source \              # the volume to backup (read-only)
  -v $(pwd):/backup \              # host directory for output
  alpine \
  tar czf /backup/pgdata-backup.tar.gz -C /source .

# Result: pgdata-backup.tar.gz in your current folder
```

**Restore a volume:**
```bash
# Create a new (empty) volume
docker volume create pgdata-restored

# Run a temporary container to extract the backup into the new volume
docker run --rm \
  -v pgdata-restored:/target \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/pgdata-backup.tar.gz -C /target

# Run your database with the restored volume
docker run -d --name pg-restored \
  -v pgdata-restored:/var/lib/postgresql/data \
  postgres:15-alpine
```

---

## 11. Common Volume Mistakes to Teach/Avoid

| Mistake | Problem | Fix |
|---|---|---|
| Not using volumes for databases | All DB data lost when container deleted | Always use named volume for DB data dir |
| Using bind mounts in production | Host path must match exactly on prod server | Use named volumes in production |
| Forgetting `-v /app/node_modules` | Container's node_modules wiped by bind mount | Add anonymous volume for node_modules |
| `docker rm -v` without knowing `-v` flag | Deletes associated anonymous volumes | Know: `rm` alone keeps volumes; `rm -v` removes anonymous ones |
| Sharing one volume between incompatible apps | Data corruption | Only share volumes between compatible, cooperative containers |

---

## 12. Complete Volume Flags Cheat Sheet

```bash
# Named volume (recommended for production)
-v my-volume:/container/path

# Bind mount (recommended for dev)
-v /absolute/host/path:/container/path
-v $(pwd):/container/path                  # current directory shortcut

# Anonymous volume (shield a path from bind mount override)
-v /container/path

# Read-only mount (container can't write to it)
-v my-volume:/container/path:ro

# tmpfs mount (RAM only, never disk)
--tmpfs /container/path
--tmpfs /container/path:size=100m          # with size limit

# Newer --mount syntax (verbose but explicit — preferred in scripts)
--mount type=volume,source=my-volume,target=/data
--mount type=bind,source=$(pwd),target=/app
--mount type=tmpfs,target=/tmp
```

---

## 13. Hands-On Practice Tasks

1. Run a PostgreSQL container **without** a volume, insert data, delete the container, restart — confirm data is gone.
2. Repeat with a named volume — confirm data survives the container deletion.
3. Run `docker volume ls` and `docker volume inspect` on your new volume.
4. Create a simple `index.html` on your host, bind-mount it into an Nginx container at `/usr/share/nginx/html`, and view it in your browser. Then edit the HTML on your host and refresh — see the change live.
5. Try the `node_modules` anonymous volume trick on any Node project.
6. Practice the backup/restore sequence from Section 10.
7. Run `docker volume prune` after removing all containers and observe which volumes are cleaned up.

---

## 14. Quiz — Test Yourself / Test Others

1. Why is container storage ephemeral by default? Where does written data go?
2. What is the key difference between a named volume and a bind mount?
3. In what scenario would you choose a named volume over a bind mount?
4. In what scenario would you choose a bind mount over a named volume?
5. What is a tmpfs mount and when should you use it?
6. What problem does `-v /app/node_modules` (anonymous volume) solve in a Node.js dev setup?
7. How do you back up a Docker volume? (Describe the approach even if you don't remember exact syntax)
8. What happens to a named volume when you run `docker rm my-container`?

<details>
<summary>Answers</summary>

1. Containers write to a thin writable layer on top of read-only image layers. This writable layer is tied to the container — when the container is deleted with `docker rm`, that layer is deleted with it.
2. A named volume is managed entirely by Docker in its internal storage area — you only reference it by name. A bind mount maps a specific path on the host filesystem that you control directly.
3. Named volume: for production data persistence (database files, uploaded files) — portable, Docker-managed, no host path dependency.
4. Bind mount: for local development — mount source code so edits on the host are instantly reflected inside the container without rebuilding the image.
5. A tmpfs mount stores data only in the host's RAM — it never touches disk and is lost when the container stops. Use for sensitive data (secrets, tokens) that must never be persisted, or for high-speed temporary scratch space.
6. When you bind mount `$(pwd):/app`, it overlays your host folder onto the container's `/app`, wiping out the container's `node_modules`. Adding `-v /app/node_modules` creates an anonymous volume that shields that specific path from the bind mount override, keeping the container's installed modules intact.
7. Spin up a temporary container that mounts both the volume (as source) and a host directory (as backup destination), then run `tar` to archive the volume contents into the host directory.
8. The named volume is NOT deleted — it persists independently. Only the container is removed. You must explicitly run `docker volume rm` or `docker volume prune` to remove it.

</details>

---

## 15. Summary

After today you can explain and implement:
- Why containers lose data by default (ephemeral writable layer)
- All three mount types: named volumes, bind mounts, tmpfs
- When to use each type (production vs development vs in-memory)
- Named volume lifecycle — create, mount, inspect, backup, restore, prune
- The `node_modules` anonymous volume trick for Node.js dev containers
- A full persistent multi-container stack (API + PostgreSQL + Redis + uploads)
- Common volume mistakes and how to avoid them

**Coming up on Day 6:** Docker Compose — the tool that takes everything you've learned this week (containers, networks, volumes) and lets you define and manage a multi-container application in a single `docker-compose.yml` file, replacing all those long `docker run` commands with one clean `docker compose up`.
