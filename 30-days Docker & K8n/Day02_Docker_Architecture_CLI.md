# Day 2: Docker Architecture Deep Dive + Core CLI Commands

**Goal for today:** Go one level deeper than Day 1 — understand *exactly* how Docker works under the hood (images, layers, the daemon's communication model), and build real fluency with the CLI commands you'll type every single day.

**Format:** WHAT → WHY → HOW for every concept, so you can teach this straight through.

---

## 0. Quick Recap (use this to open your teaching session)

Yesterday you learned:
- Containers solve the "works on my machine" problem better than VMs (lighter, faster, share host kernel)
- Docker = Client + Daemon + Registry
- Image = blueprint, Container = running instance of that blueprint
- Basic commands: `run`, `ps`, `stop`, `rm`

Today we open up the hood and look at *how* images are actually built and *how* the daemon really talks to things — then we drill the CLI commands until they're muscle memory.

---

## 1. Docker Architecture — Going Deeper

### WHAT
Yesterday you saw: Client → Daemon → Registry. Today, the precise mechanics:

- The **Docker Client** doesn't talk to the daemon "magically" — it sends REST API calls over a **Unix socket** (on Linux/Mac: `/var/run/docker.sock`) or a network socket.
- The **Docker Daemon (dockerd)** listens for these API requests and is the one process that actually has permission to create namespaces, cgroups, manage images, networks, and volumes.
- Docker objects the daemon manages: **Images, Containers, Networks, Volumes** (you've met the first two; Networks and Volumes come later this week).

### WHY
Understanding this matters because:
- It explains why you often need elevated permissions to run Docker (the daemon needs deep OS access)
- It explains why tools like Docker Desktop, CI/CD pipelines, and remote Docker hosts can all use the *same* CLI — they're just pointing the client at a different daemon socket/address
- It's the foundation for understanding Kubernetes later (Kubernetes also has a client → API-server → worker-node pattern — same architectural philosophy)

### HOW (the request lifecycle, teach this flow)
```
$ docker run -d -p 8080:80 nginx

1. Client parses the command into a REST API request
2. Request sent over Unix socket to dockerd
3. Daemon checks local image cache for "nginx"
      │
      ├── Found locally → skip to step 5
      │
      └── Not found → daemon contacts the Registry (Docker Hub)
4. Registry sends the image layers back; daemon stores them locally
5. Daemon creates a container (allocates namespaces + cgroups)
6. Daemon sets up networking (the -p 8080:80 mapping)
7. Daemon starts the container process
8. Daemon returns status to Client → you see container ID printed
```

**Teaching line:**
> "Every Docker command you type is really just a REST API call in disguise. The CLI is a thin client — all the real intelligence lives in the daemon."

---

## 2. Images vs Containers — The Deep Dive (Layers)

### WHAT
An **image** is not one single file — it's a stack of **read-only layers**, each representing a change (e.g., "install Python," "copy app code," "set environment variable"). Docker uses a **Union File System** to stack these layers and present them as a single unified filesystem.

A **container** = all those read-only image layers + **one thin writable layer** added on top, specific to that running container.

### WHY
This layered design is *the* core efficiency trick of Docker:
- **Layer caching**: if you rebuild an image and only the last instruction changed, Docker reuses all the earlier unchanged layers instead of rebuilding from scratch (huge build-speed win — you'll feel this directly on Day 3)
- **Storage efficiency**: if 10 containers all use the same base image (e.g., `ubuntu`), that base layer is stored **once** on disk and shared — not duplicated 10 times
- **Disposability**: because containers only add a thin writable layer, deleting a container and starting fresh is cheap and fast — this is *why* containers are treated as disposable/ephemeral by design

### HOW (visualize it)
```
┌─────────────────────────────────┐
│   Container's Writable Layer      │  ← unique per container, deleted when container is removed
├─────────────────────────────────┤
│   Image Layer: "COPY app code"    │  ┐
├─────────────────────────────────┤  │  Read-only,
│   Image Layer: "RUN npm install"  │  │  shared across
├─────────────────────────────────┤  │  every container
│   Image Layer: "Install Node"     │  │  built from this
├─────────────────────────────────┤  │  image
│   Base Layer: "Ubuntu OS files"   │  ┘
└─────────────────────────────────┘
```

**Teaching analogy:**
> "Think of layers like transparent sheets stacked on an overhead projector. Each sheet (layer) adds something, but you only see the combined picture. The container just adds one more sheet on top that you're allowed to write on — the sheets underneath stay untouched."

---

## 3. The Four Core Docker Objects (preview)

| Object | What it is | When we cover it deeply |
|---|---|---|
| **Image** | Blueprint built from layers | Day 3 (Dockerfile) |
| **Container** | Running instance of an image | Today |
| **Network** | How containers talk to each other / outside world | Day 4 |
| **Volume** | Persistent data storage outside the container's writable layer | Day 5 |

Knowing this map helps a student see *where* today's topic fits into the bigger picture.

---

## 4. Core CLI Commands — Build Real Fluency

This is the heart of today. For each command: **what it does, why you'd use it, exact syntax, example.**

### 4.1 `docker run` — create + start a container

**WHAT:** Creates a new container from an image and starts it.
**WHY:** It's the single most-used command — your entry point to running anything.

**Key flags (memorize these):**
| Flag | Meaning |
|---|---|
| `-d` | Detached mode (run in background) |
| `-it` | Interactive + TTY (attach your terminal) |
| `--name <name>` | Give the container a friendly name instead of a random one |
| `-p host:container` | Map a host port to a container port |
| `-e KEY=value` | Set an environment variable inside the container |
| `-v host:container` | Mount a volume/bind path (Day 5 topic, but you'll see it used) |
| `--rm` | Auto-delete the container when it stops (great for throwaway testing) |
| `--network <name>` | Attach to a specific network (Day 4 topic) |

**Example:**
```bash
docker run -d --name my-web -p 8080:80 -e ENV=production --rm nginx
```
> "Run nginx in the background, name it my-web, map port 8080→80, set an ENV variable, and auto-clean-up when stopped."

---

### 4.2 `docker ps` — list containers

**WHAT:** Lists containers.
**WHY:** You need to see what's running (or what crashed) before you can manage it.

```bash
docker ps          # running containers only
docker ps -a       # all containers, including stopped/exited
docker ps -q       # only container IDs (useful for scripting)
```

---

### 4.3 `docker exec` — run a command inside an *already running* container

**WHAT:** Executes a new command inside a container that's already running.
**WHY:** This is different from `docker run` — `run` creates a *new* container from an image; `exec` lets you peek into or interact with a container that's *already alive*. This distinction trips up almost every beginner, so always teach it explicitly.

```bash
docker exec -it my-web bash      # open a shell inside the running container
docker exec my-web ls /app       # run one command and see the output
```

**Teaching line (this comparison is critical):**
> "`docker run` is like hiring a brand-new employee and giving them a task. `docker exec` is like walking up to an employee who's already working and asking them to do one more thing."

---

### 4.4 `docker logs` — see a container's output

**WHAT:** Shows the stdout/stderr output of a container.
**WHY:** Essential for debugging — if your app isn't behaving, logs are the first place you look.

```bash
docker logs my-web           # show all logs so far
docker logs -f my-web        # follow logs live (like `tail -f`)
docker logs --tail 50 my-web # only last 50 lines
```

---

### 4.5 Lifecycle commands: `stop`, `start`, `restart`, `rm`

| Command | What it does |
|---|---|
| `docker stop <name/id>` | Gracefully stops a running container |
| `docker start <name/id>` | Starts a previously stopped container (reuses it, doesn't create new) |
| `docker restart <name/id>` | Stop + start in one command |
| `docker rm <name/id>` | Removes a stopped container permanently |
| `docker rm -f <name/id>` | Force-removes a container even if running |

**Teaching nuance:** `stop` ≠ `rm`. Stopping keeps the container (and its writable layer / any state) intact so you can `start` it again later. `rm` deletes it permanently.

---

### 4.6 Image-related commands

| Command | What it does |
|---|---|
| `docker images` | Lists all images stored locally |
| `docker pull <image>` | Downloads an image from a registry without running it |
| `docker rmi <image>` | Removes an image from local storage |

```bash
docker pull node:18
docker images
docker rmi node:18
```

---

### 4.7 `docker inspect` — see the full details of an object

**WHAT:** Returns a detailed JSON dump of a container's (or image's) configuration — IP address, mounted volumes, env vars, network settings, everything.
**WHY:** When `logs` and `ps` aren't enough, `inspect` is your ground truth for "what exactly is this container configured to do?"

```bash
docker inspect my-web
docker inspect my-web | grep IPAddress    # pull out just one detail
```

---

### 4.8 `docker cp` — copy files in/out of a container

```bash
docker cp my-web:/app/log.txt ./log.txt     # container → host
docker cp ./config.json my-web:/app/        # host → container
```

---

## 5. Hands-On Practice Tasks

1. Run `docker run -d --name practice -p 8081:80 nginx`
2. Confirm it's running with `docker ps`
3. Use `docker exec -it practice bash` to go inside, run `ls /usr/share/nginx/html`, then `exit`
4. View logs with `docker logs practice`
5. Stop it: `docker stop practice` → confirm with `docker ps -a` (still listed, but not running)
6. Start it again: `docker start practice` → confirm it's running again
7. Run `docker inspect practice` and find its internal IP address
8. Finally remove it: `docker rm -f practice`
9. Run `docker pull redis` then `docker images` to see it listed without ever running it

---

## 6. Cheat Sheet (your revision sheet for today)

```
CONTAINER LIFECYCLE
docker run [-d] [-it] [--name x] [-p host:container] image   → create + start
docker exec -it <name> bash                                   → enter running container
docker logs [-f] <name>                                       → view output
docker stop/start/restart <name>                               → lifecycle control
docker rm [-f] <name>                                          → delete container

INSPECTION
docker ps [-a] [-q]        → list containers
docker inspect <name>      → full JSON details

IMAGES
docker images               → list local images
docker pull <image>:<tag>   → download image only
docker rmi <image>          → delete image

FILES
docker cp <src> <dest>      → copy files in/out
```

---

## 7. Quiz — Test Yourself / Test Others

1. What protocol/mechanism does the Docker Client actually use to talk to the Daemon?
2. What is a Union File System, and why does Docker use one?
3. What's the key difference between `docker run` and `docker exec`?
4. If you `docker stop` a container, is its data lost? What about `docker rm`?
5. Why is layer caching valuable when building images?
6. Which command would you use to see a container's internal IP address?
7. What does `docker ps -a` show that plain `docker ps` does not?

<details>
<summary>Answers</summary>

1. REST API calls sent over a Unix socket (or network socket).
2. A filesystem that stacks read-only layers into one unified view; Docker uses it so images can share layers and containers only need a thin writable layer on top — saving space and speeding up builds.
3. `run` creates a brand-new container from an image; `exec` runs a command inside a container that's already running.
4. `stop` keeps the container and its data intact (can be restarted); `rm` permanently deletes the container and its writable layer.
5. Unchanged layers don't need to be rebuilt, so only the modified/new layers are processed — much faster builds.
6. `docker inspect <name>` (look for the IPAddress field).
7. `docker ps -a` also shows stopped/exited containers, not just running ones.

</details>

---

## 8. Summary

After today you should be able to explain:
- The exact request lifecycle from CLI command → Daemon → Registry → running container
- Why images are layered, and how that layering makes Docker fast and storage-efficient
- The Image/Container/Network/Volume object map
- The full core CLI toolkit: `run`, `ps`, `exec`, `logs`, `stop/start/restart`, `rm`, `images`, `pull`, `rmi`, `inspect`, `cp`
- The critical `run` vs `exec` distinction (the #1 beginner confusion)

**Coming up on Day 3:** The Dockerfile — how to actually *build* your own custom images from scratch (instructions, layer caching in practice, and packaging a real Node.js/Express app, which connects directly to your own QueueCare/PulseBloom stack).
