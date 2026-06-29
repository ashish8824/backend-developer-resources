# Day 4: Docker Networking — How Containers Talk to Each Other & the World

**Goal for today:** Understand how Docker's network system works, the different network driver types (bridge, host, none, overlay), how containers discover and talk to each other using DNS, and how traffic flows in/out of containers. By the end, you should be able to set up multi-container communication from scratch and explain every networking concept clearly.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 1: Containers = lightweight isolated environments sharing host OS kernel
- Day 2: Core CLI — `run`, `exec`, `logs`, `inspect`, `ps`
- Day 3: Dockerfile — building custom images, layer caching, `.dockerignore`, multi-stage builds

**Today's connecting thought:**
> So far, every container we ran was essentially an island — isolated by design. But real apps are never a single isolated process. Your Express API needs to talk to a PostgreSQL database. Your frontend needs to reach your backend. Today we build the bridges between those islands.

---

## 1. The Problem — Why Container Networking Needs Special Attention

**Scenario:**
You run two containers:
```bash
docker run -d --name api node-api       # your Express API
docker run -d --name db postgres        # your PostgreSQL database
```

**Problem:** Can the `api` container reach the `db` container?

The answer is: **it depends on how they're networked.** Unlike two processes on the same machine sharing `localhost`, containers are isolated. They need an explicit network path to communicate.

This is what Docker networking solves.

---

## 2. How Docker Networking Works — The Mental Model

### WHAT
Docker creates **virtual networks** — software-defined networks that containers can be attached to. When two containers are on the same Docker network, they can reach each other by **container name** (Docker has built-in DNS for this). When they're on different networks, they cannot reach each other by default.

### WHY
This isolation-by-default is a security feature:
- Your payment service and your analytics service shouldn't be able to talk to each other unless explicitly allowed
- An attacker who compromises one container can't automatically reach all other containers on the machine
- You define communication paths intentionally, not accidentally

### HOW — The Big Picture
```
┌─────────────────────────────────────────────────────────────┐
│                       Host Machine                           │
│                                                              │
│   ┌──────────── Docker Bridge Network (my-app-net) ───────┐  │
│   │                                                         │  │
│   │  ┌─────────────┐        ┌─────────────────┐            │  │
│   │  │  Container:  │        │   Container:     │            │  │
│   │  │  api         │◄──────►│   db (postgres)  │            │  │
│   │  │  172.18.0.2  │  DNS   │   172.18.0.3     │            │  │
│   │  └─────────────┘        └─────────────────┘            │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                              │
│   ┌──────────── Docker Bridge Network (default) ──────────┐  │
│   │                                                         │  │
│   │  ┌─────────────┐                                        │  │
│   │  │  Container:  │ (CANNOT reach api or db above)        │  │
│   │  │  other-app   │                                        │  │
│   │  └─────────────┘                                        │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Docker Network Drivers — The Four Types

Docker uses a concept called **network drivers** to implement different networking behaviours. Think of drivers as different "modes" of networking.

---

### 3.1 Bridge Network (default, most used)

#### WHAT
A **bridge network** creates a private internal network on the host. Containers connected to the same bridge network can communicate with each other. Containers on different bridge networks are isolated.

#### WHY
This is the default and most common driver — used for most single-host multi-container setups. It gives you isolation between app stacks while allowing communication within a stack.

#### HOW

**The default bridge network (automatic):**
```bash
docker run -d --name container1 nginx
docker run -d --name container2 nginx
```
Both containers are automatically on Docker's default bridge network called `bridge`. BUT — on the default bridge, containers can only reach each other by **IP address**, not by name. This makes it fragile.

**Custom bridge network (what you should always use):**
```bash
# Create a named custom bridge network
docker network create my-app-net

# Run containers and attach them to your custom network
docker run -d --name api --network my-app-net my-node-api
docker run -d --name db --network my-app-net postgres

# Now api can reach db using the name "db"!
# From inside api container:
# → ping db          (works!)
# → psql -h db ...   (works!)
```

**The key difference — custom bridge gives you DNS:**
| Feature | Default Bridge | Custom Bridge |
|---|---|---|
| Container-to-container communication | By IP only | By container name (DNS) |
| Automatic DNS resolution | ❌ No | ✅ Yes |
| Isolation from other networks | Partial | Full |
| Recommended for production | ❌ No | ✅ Yes |

**Teaching line:**
> "The default bridge is like a street where houses have no addresses — you can only visit if you already know the exact GPS coordinates. A custom bridge is like a named street — you just say 'go to the house named db' and Docker's built-in GPS (DNS) figures it out."

---

### 3.2 Host Network

#### WHAT
The container shares the **host machine's network stack** directly — no isolation, no virtual network interface. The container uses the host's IP and ports as if it were a regular process running directly on the host.

#### WHY
Maximum network performance — no virtual network overhead. Used when a container needs bare-metal network performance (e.g., high-throughput packet processing, some monitoring agents).

#### HOW
```bash
docker run -d --network host nginx
```
- Nginx inside the container binds to port 80 on the host directly
- You don't need `-p 80:80` — the container IS the host, network-wise
- On Mac and Windows, `--network host` doesn't work the same way (Linux-only feature)

**Trade-off:**
- ✅ Best network performance
- ❌ No port isolation — if the container uses port 3000, it occupies YOUR host's port 3000
- ❌ Less secure — container can see all host network interfaces

**Teaching line:**
> "Host network mode removes the wall between the container and the host's network. It's like moving from a private apartment (bridge) to just sleeping in the lobby — faster access, but no privacy."

---

### 3.3 None Network

#### WHAT
The container gets **no network interface at all** — completely disconnected from everything. Not even a loopback interface.

#### WHY
Maximum isolation for containers that should never touch a network — batch processing jobs, offline data transformations, security-sensitive computations.

#### HOW
```bash
docker run -d --network none my-batch-job
```
This container cannot send or receive any network traffic whatsoever.

**Teaching line:**
> "None network is airplane mode for containers. You use it when you want to guarantee the container never phones home."

---

### 3.4 Overlay Network (preview — important for Kubernetes week)

#### WHAT
An **overlay network** spans **multiple Docker hosts** (multiple physical/virtual machines). Containers on different hosts but the same overlay network can communicate as if they were on the same local network.

#### WHY
This is how Docker Swarm (and Kubernetes) enable container communication across a whole cluster of machines. You'll use this concept heavily from Day 11 onward.

#### HOW (conceptual for now)
```
Host Machine 1          Host Machine 2
┌──────────────┐        ┌──────────────┐
│  Container A  │◄──────►│  Container B  │
│  (10.0.0.2)  │ Overlay │  (10.0.0.3)  │
└──────────────┘ Network └──────────────┘
```
Actual traffic is **encapsulated (tunneled)** between hosts — containers see a flat network, unaware they're on different machines.

---

## 4. Network CLI Commands — Full Reference

```bash
# List all networks
docker network ls

# Create a custom bridge network
docker network create my-app-net

# Create with specific subnet
docker network create --subnet=192.168.10.0/24 my-app-net

# Inspect a network (see connected containers, IP ranges, driver)
docker network inspect my-app-net

# Connect a running container to a network
docker network connect my-app-net my-container

# Disconnect a container from a network
docker network disconnect my-app-net my-container

# Remove a network (only if no containers are using it)
docker network rm my-app-net

# Remove all unused networks
docker network prune
```

---

## 5. Port Mapping — Deep Dive

### WHAT
Port mapping (also called port publishing) creates a rule that forwards traffic from a **host port** to a **container port**. This is how external traffic (your browser, Postman, curl) reaches inside a container.

### WHY
Containers are isolated — they have their own network namespace. Port 3000 inside a container is NOT the same as port 3000 on your host. Port mapping punches a specific hole through that isolation.

### HOW

**Syntax:**
```bash
docker run -p <host_port>:<container_port> image
```

**Examples:**
```bash
docker run -p 3000:3000 my-api          # host 3000 → container 3000 (same)
docker run -p 8080:3000 my-api          # host 8080 → container 3000 (different)
docker run -p 127.0.0.1:3000:3000 api   # only localhost (not LAN-accessible)
docker run -p 3000:3000 -p 5000:5000 my-app  # multiple ports
docker run -P my-app                     # auto-assign random host ports
```

**Visualize the traffic flow:**
```
Your browser → localhost:8080
                    │
                    ▼ (port mapping rule)
              Host Machine
                    │
                    ▼
         Docker virtual bridge
                    │
                    ▼
         Container's port 3000
                    │
                    ▼
         Express app listening on :3000
```

**Teaching line:**
> "Port mapping is like a hotel switchboard. You dial room extension 8080 (host port), the switchboard redirects your call to the internal room 3000 (container port). The room doesn't know you called 8080 — from its perspective, you always call 3000."

---

## 6. Container DNS — How Containers Find Each Other by Name

### WHAT
Docker runs a built-in **DNS server** for every custom bridge network. When a container needs to find another container by name, it asks Docker's DNS, which resolves the name to the target container's IP address on that network.

### WHY
Container IP addresses are **dynamic** — they're assigned when a container starts and can change if the container restarts. Hardcoding IPs in your app config would break constantly. DNS lets you use stable names (`db`, `redis`, `api`) that always point to the right container regardless of what IP they have.

### HOW

**In your app code, connect using the container name:**
```javascript
// In your Express API (Node.js) - connecting to PostgreSQL container named "db"
const pool = new Pool({
  host: 'db',        // ← container name, Docker DNS resolves this
  port: 5432,
  database: 'queuecare',
  user: 'postgres',
  password: 'secret'
});
```

```bash
# Setup
docker network create queuecare-net

docker run -d \
  --name db \
  --network queuecare-net \
  -e POSTGRES_DB=queuecare \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

docker run -d \
  --name api \
  --network queuecare-net \
  -p 3000:3000 \
  queuecare-api:1.0

# From inside api container:
docker exec -it api ping db      # ✅ resolves! Docker DNS translates "db" → 172.18.0.3
```

---

## 7. Connecting a Container to Multiple Networks

### WHAT
A single container can be attached to **more than one** network at the same time. It gets a separate IP on each network it's part of.

### WHY
Real microservice apps often have multiple network "zones":
- A **frontend network**: frontend container + API container (frontend talks to API)
- A **backend network**: API container + DB container (API talks to DB)
- The DB should NOT be on the frontend network (frontend shouldn't reach DB directly)

### HOW
```bash
docker network create frontend-net
docker network create backend-net

# API is on BOTH networks (it's the bridge between frontend and backend)
docker run -d --name api --network frontend-net my-api
docker network connect backend-net api

# Frontend only on frontend-net (can reach api, NOT db)
docker run -d --name frontend --network frontend-net my-frontend

# DB only on backend-net (can reach api, NOT frontend)
docker run -d --name db --network backend-net postgres
```

**Architecture visualized:**
```
frontend-net:                    backend-net:
┌─────────────────┐              ┌─────────────────┐
│  frontend ◄────►│ api ◄───────►│ db               │
└─────────────────┘              └─────────────────┘
                   ↑
              (connected to both networks)
```

---

## 8. Real-World Example — Node.js API + PostgreSQL + Redis

This is the pattern you'd use for something like your QueueCare or PulseBloom stack manually (before we use Docker Compose on Day 6 to automate this).

```bash
# Step 1: Create isolated network
docker network create pulsebloom-net

# Step 2: Start PostgreSQL
docker run -d \
  --name postgres \
  --network pulsebloom-net \
  -e POSTGRES_USER=pulsebloom \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=pulsebloom \
  postgres:15-alpine

# Step 3: Start Redis
docker run -d \
  --name redis \
  --network pulsebloom-net \
  redis:7-alpine

# Step 4: Start your API (can reach 'postgres' and 'redis' by name)
docker run -d \
  --name api \
  --network pulsebloom-net \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://pulsebloom:secret123@postgres:5432/pulsebloom \
  -e REDIS_URL=redis://redis:6379 \
  pulsebloom-api:1.0

# Verify everything can communicate
docker exec -it api ping postgres    # ✅
docker exec -it api ping redis       # ✅
```

Notice: `postgres` and `redis` in the URLs above are **container names** — Docker DNS resolves them automatically.

---

## 9. Inspecting Networks — Debugging Networking Issues

When containers can't talk to each other, here's your debugging toolkit:

```bash
# 1. Confirm both containers are on the same network
docker network inspect my-app-net

# 2. Check a container's network config
docker inspect api | grep -A 20 '"Networks"'

# 3. Test DNS resolution from inside a container
docker exec -it api ping db
docker exec -it api nslookup db

# 4. Test port connectivity from inside a container
docker exec -it api wget -qO- http://db:5432
docker exec -it api curl http://other-service:3000/health

# 5. List all networks a container is on
docker inspect api --format='{{json .NetworkSettings.Networks}}'
```

**Common networking problems and fixes:**

| Problem | Cause | Fix |
|---|---|---|
| `ping db` fails | Containers on different networks | Put them on the same custom bridge |
| `ping db` fails | Using default bridge, not custom | Create a custom network |
| `ping db` works but app can't connect | Wrong port / service not ready | Check container logs |
| Can't access container from browser | No port mapping | Add `-p host:container` |
| Port mapping works locally not in prod | Binding to `127.0.0.1` | Bind to `0.0.0.0` or remove IP restriction |

---

## 10. Network Drivers — Quick Comparison Table

| Driver | Scope | DNS by Name | Use Case |
|---|---|---|---|
| `bridge` (default) | Single host | ❌ No | Quick testing |
| `bridge` (custom) | Single host | ✅ Yes | App stacks on one host |
| `host` | Single host | N/A (shares host) | High-performance / monitoring |
| `none` | Single host | ❌ No | Completely isolated jobs |
| `overlay` | Multi-host | ✅ Yes | Swarm / Kubernetes clusters |

---

## 11. Hands-On Practice Tasks

1. Run `docker network ls` and observe which networks already exist by default (`bridge`, `host`, `none`)
2. Create a custom network: `docker network create practice-net`
3. Run two containers on it:
   ```bash
   docker run -d --name container1 --network practice-net nginx
   docker run -d --name container2 --network practice-net alpine sleep 3600
   ```
4. From container2, ping container1 by name:
   ```bash
   docker exec -it container2 ping container1
   ```
5. Run `docker network inspect practice-net` and see both containers listed
6. Disconnect container2 from the network and retry the ping — confirm it fails
7. Set up the 3-container stack from Section 8 (PostgreSQL + Redis + API) if you have an app ready, or just set up PostgreSQL + ping it from an alpine container

---

## 12. Quiz — Test Yourself / Test Others

1. What is the key difference between the **default** bridge network and a **custom** bridge network?
2. Why are container IPs unreliable for inter-container communication? What should you use instead?
3. What does `--network host` mean? What do you gain and lose?
4. In `docker run -p 8080:3000`, which port does the app inside the container actually listen on?
5. What Docker feature allows `host: 'db'` in your app config to resolve correctly?
6. Why might you connect a container to two different networks?
7. A developer tells you their two containers can't communicate. What's the first thing you'd check?

<details>
<summary>Answers</summary>

1. The default bridge allows container-to-container communication only by IP address (no name resolution). A custom bridge network provides built-in DNS — containers can reach each other using their container names.
2. Container IPs are dynamically assigned and can change on restarts. You should use container names instead, which Docker DNS always resolves to the correct current IP.
3. `--network host` makes the container share the host machine's full network stack directly (no virtual network). You gain maximum network performance, but lose port isolation and network security.
4. The app listens on port 3000 (the right side of the mapping). The host exposes 8080 externally and forwards traffic to container port 3000.
5. Docker's built-in DNS server, available on all custom bridge networks. It resolves container names to their current IPs automatically.
6. To create network isolation zones — e.g., an API container needs to talk to both the frontend (on frontend-net) and the database (on backend-net), but the frontend should not have direct access to the database.
7. Check whether both containers are on the same custom bridge network using `docker network inspect`. Most communication failures come from containers being on different or default networks.

</details>

---

## 13. Summary

After today you can explain and implement:
- Why Docker needs its own networking layer (container isolation)
- All 4 network drivers: bridge (default vs custom), host, none, overlay
- How Docker's built-in DNS works and why it's essential
- Port mapping: syntax, traffic flow, real-world usage
- Multi-network architectures for proper service isolation
- The full CLI: `network create/ls/inspect/connect/disconnect/rm/prune`
- How to debug networking issues step by step

**Coming up on Day 5:** Docker Volumes — the last major Docker primitive. How to make container data persist across restarts and container deletions, the difference between named volumes and bind mounts, and why stateful apps (like your PostgreSQL database) need volumes to survive.
