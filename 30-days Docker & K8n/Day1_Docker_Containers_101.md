# Day 1: Containers 101 — The Foundation

**Goal for today:** Understand *why* containers exist, *what* Docker is, and run your very first container. By the end, you should be able to teach someone else this entire day from memory.

**Format:** Every concept follows **WHAT → WHY → HOW** so you can explain it the same way to a student.

---

## 1. The Problem Docker Solves (Start here when teaching)

Before explaining Docker, always hook your audience with the problem first.

### The "Works on My Machine" Problem

**Scenario:** A developer builds an app on their laptop (Node v18, specific OS libraries, a certain database version). It works perfectly. They send it to a teammate or deploy it to a server — and it breaks.

**Why it breaks:**
- Different OS (Windows vs Linux vs Mac)
- Different versions of Node, Python, Java
- Missing system libraries
- Different environment variables / configs
- "It works on my machine" becomes a running joke in every dev team

**Teaching line to use:**
> "Software doesn't run in isolation — it depends on its environment. If the environment changes, the software can break, even if the code never changed."

This is the exact problem Docker was built to solve: **package the application together with everything it needs to run, so it behaves identically everywhere.**

---

## 2. What is Virtualization (the old solution)?

### WHAT
Virtualization means running a full **virtual computer (Virtual Machine / VM)** inside your physical computer. Each VM has its own complete operating system (kernel, drivers, everything), even though it's sharing the same physical hardware.

### WHY
Before containers, this was the standard way to isolate applications:
- You wanted to run 3 different apps on 1 physical server without them interfering with each other
- Each app got its own VM with its own OS, so they couldn't clash

### HOW (conceptually)
A piece of software called a **Hypervisor** (e.g., VMware, VirtualBox, Hyper-V) sits on top of physical hardware and creates multiple virtual machines, each with:
- Its own full OS (Guest OS)
- Its own virtual CPU, RAM, disk
- Complete isolation from other VMs

```
┌─────────────────────────────────────────────┐
│              Physical Server (Host)          │
│  ┌──────────────────────────────────────┐    │
│  │            Hypervisor                  │   │
│  │  ┌───────────┐ ┌───────────┐ ┌──────┐ │   │
│  │  │   VM 1     │ │   VM 2     │ │ VM 3 │ │   │
│  │  │ Guest OS   │ │ Guest OS   │ │ ...  │ │   │
│  │  │ App A      │ │ App B      │ │      │ │   │
│  │  └───────────┘ └───────────┘ └──────┘ │   │
│  └──────────────────────────────────────┘    │
│              Host OS + Hardware               │
└─────────────────────────────────────────────┘
```

### The Problem with VMs
- Each VM carries a **full OS** → heavy (GBs in size)
- Slow to boot (minutes)
- Wastes CPU/RAM running duplicate OS kernels
- If you need to run 10 small apps, you're running 10 full operating systems just to isolate them

**Teaching line:**
> "A VM is like giving every employee their own office building when all they needed was their own desk."

---

## 3. What are Containers?

### WHAT
A **container** is a lightweight, isolated environment for running an application — but unlike a VM, it does **not** include a full OS. It shares the host machine's OS kernel and only packages the application code, libraries, and dependencies it actually needs.

### WHY
Containers solve the VM's weight problem while still solving the original "works on my machine" problem:
- **Lightweight**: MBs instead of GBs (no duplicate OS)
- **Fast**: starts in milliseconds/seconds, not minutes
- **Consistent**: same container runs identically on your laptop, your teammate's laptop, and the production server
- **Efficient**: you can run dozens of containers on a machine that could only run a few VMs

### HOW (conceptually)
Containers use OS-level features (on Linux: **namespaces** for isolation and **cgroups** for resource limits) to isolate processes from each other while all sharing one underlying OS kernel.

```
┌─────────────────────────────────────────────┐
│              Physical/Virtual Server          │
│  ┌──────────────────────────────────────┐    │
│  │         Container Engine (Docker)      │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │   │
│  │  │Container1│ │Container2│ │Container3│  │   │
│  │  │ App A    │ │ App B    │ │ App C    │  │   │
│  │  │ + libs   │ │ + libs   │ │ + libs   │  │   │
│  │  └─────────┘ └─────────┘ └─────────┘  │   │
│  └──────────────────────────────────────┘    │
│            Single Shared Host OS Kernel        │
│                   Hardware                     │
└─────────────────────────────────────────────┘
```

**Teaching line:**
> "A container is like giving every employee their own desk in a shared office building — they each have their own space, tools, and privacy, but they share the building's plumbing, electricity, and structure (the OS kernel)."

---

## 4. VM vs Container — The Comparison Table (great for teaching)

| Aspect | Virtual Machine | Container |
|---|---|---|
| OS | Full guest OS per VM | Shares host OS kernel |
| Size | GBs | MBs |
| Boot time | Minutes | Seconds/milliseconds |
| Isolation | Strong (hardware-level) | Process-level (slightly less strong) |
| Performance | Heavier, slower | Near-native speed |
| Density | Few per host | Dozens-hundreds per host |
| Use case | Run different OS types | Run same-OS-family apps fast & consistently |

**Important nuance to teach:** Containers and VMs aren't always "either/or" — in production, you'll often see **containers running inside VMs** (e.g., a Kubernetes node is often a VM, and it runs many containers inside it). This gives both strong isolation (VM boundary) and efficiency (containers inside).

---

## 5. What is Docker?

### WHAT
Docker is the most popular **platform/toolset** for building, running, and managing containers. It's not the only way to make containers (the underlying Linux features existed before Docker), but Docker made containers easy to use for everyday developers.

### WHY
Before Docker (2013), containers existed but were hard to use — you needed deep Linux kernel knowledge. Docker gave developers:
- A simple CLI (`docker run`, `docker build`, etc.)
- A standard packaging format (the **image**)
- A way to share images via **registries** (like Docker Hub — think "GitHub for container images")
- Tools to build, ship, and run containers consistently

### HOW (the big picture)
Docker has 3 core building blocks you must memorize and be able to explain:

1. **Dockerfile** — a text file with instructions for *building* an image (we cover this Day 3)
2. **Image** — a read-only, packaged snapshot of an application + its dependencies (built from a Dockerfile)
3. **Container** — a *running instance* of an image

**Teaching analogy (use this — it sticks):**
> "Think of an **Image** as a recipe/blueprint (like a class in programming), and a **Container** as the actual running thing made from that blueprint (like an object/instance). You can create many containers from one image, just like you can create many objects from one class."

---

## 6. Docker Architecture (high-level, deeper dive on Day 2)

### WHAT
Docker uses a **client-server architecture**:
- **Docker Client** — the CLI you type commands into (`docker run ...`)
- **Docker Daemon (dockerd)** — a background service that does the actual work (building images, running containers, managing networks)
- **Docker Registry** — a storage/distribution service for images (Docker Hub is the default public one)

### WHY
This separation lets the client talk to a daemon running locally *or* remotely, and lets images be shared globally via registries instead of everyone rebuilding from scratch.

### HOW
```
You type:  docker run nginx
              │
              ▼
     ┌─────────────────┐
     │  Docker Client    │  (CLI)
     └────────┬─────────┘
              │ REST API call
              ▼
     ┌─────────────────┐
     │  Docker Daemon    │  (does the real work)
     │   (dockerd)       │
     └────────┬─────────┘
              │ "Do I have the nginx image locally?"
              │ No → pull it
              ▼
     ┌─────────────────┐
     │  Docker Hub        │  (Registry)
     │  (image storage)   │
     └─────────────────┘
              │
              ▼
     Daemon creates and runs
     the container from the image
```

---

## 7. Installing Docker

### WHAT
Docker Desktop (Windows/Mac) or Docker Engine (Linux) — the software that gives you the client + daemon.

### HOW
- **Windows/Mac:** Download Docker Desktop from docker.com, install, and it includes everything (client, daemon, GUI).
- **Linux:** Install Docker Engine via your package manager (e.g., `apt` on Ubuntu).

### Verify installation
```bash
docker --version
docker info
```
- `docker --version` → confirms Docker CLI is installed
- `docker info` → shows daemon status, confirms the daemon is actually running (not just the CLI)

**Teaching tip:** If `docker --version` works but `docker info` fails, that means the *client* is installed but the *daemon* isn't running — a very common beginner confusion. This reinforces the client/daemon distinction from Section 6.

---

## 8. Running Your First Container

### Command 1: The "Hello World" of Docker
```bash
docker run hello-world
```

**What happens step by step (explain this exact flow to students):**
1. Docker Client sends the `run` command to the Daemon
2. Daemon checks: "Do I have the `hello-world` image locally?" → No (first time)
3. Daemon pulls the image from Docker Hub (the registry)
4. Daemon creates a container from that image
5. Daemon runs it — the container prints a message and exits
6. You see the output in your terminal

This single command demonstrates the *entire* architecture from Section 6 in action.

### Command 2: Run an interactive container
```bash
docker run -it ubuntu bash
```
- `-it` = interactive + terminal (lets you type commands inside the container, like SSH-ing into a tiny machine)
- `ubuntu` = the image name (a minimal Ubuntu Linux image)
- `bash` = the command to run inside the container (opens a shell)

Try running `ls`, `pwd`, `whoami` inside it — you're now inside an isolated Linux environment. Type `exit` to leave.

### Command 3: See it running in the background
```bash
docker run -d -p 8080:80 nginx
```
- `-d` = detached mode (runs in background, doesn't block your terminal)
- `-p 8080:80` = port mapping → **host port 8080 → container port 80**
- `nginx` = a popular web server image

Now open a browser to `http://localhost:8080` — you'll see the Nginx welcome page, served from inside a container.

**Teaching line for port mapping (commonly confusing):**
> "The container is isolated and has its own internal network. Port mapping is like punching a hole through the wall so traffic from your laptop's port 8080 can reach the container's internal port 80."

### Check what's running
```bash
docker ps          # shows running containers
docker ps -a       # shows ALL containers, including stopped ones
```

### Stop and remove
```bash
docker stop <container_id>
docker rm <container_id>
```

---

## 9. Key Vocabulary Cheat Sheet (use this as your revision sheet)

| Term | One-line definition |
|---|---|
| **Image** | Read-only blueprint/template for a container |
| **Container** | A running (or stopped) instance of an image |
| **Dockerfile** | Instructions to build an image (Day 3 topic) |
| **Registry** | A place to store/share images (e.g., Docker Hub) |
| **Docker Daemon** | Background service that does the actual container work |
| **Docker Client** | The CLI you interact with |
| **Port mapping** | Connects a host port to a container's internal port |

---

## 10. Hands-On Practice Tasks (do these before Day 2)

1. Install Docker and confirm with `docker --version` and `docker info`
2. Run `docker run hello-world` and read the output message carefully — it actually explains the architecture flow itself
3. Run `docker run -it ubuntu bash`, explore with `ls`/`pwd`, then `exit`
4. Run `docker run -d -p 8080:80 nginx` and view it in your browser
5. Run `docker ps` and `docker ps -a` — note the difference
6. Stop and remove the nginx container

---

## 11. Quiz — Test Yourself (and use these to teach/test others)

1. What is the main architectural difference between a VM and a container?
2. Why are containers faster to start than VMs?
3. What is the relationship between an Image and a Container? (Use the class/object analogy)
4. What are the 3 main components of Docker's architecture?
5. In `docker run -d -p 8080:80 nginx`, what does each flag mean?
6. What command shows only currently running containers? What command shows all containers including stopped ones?
7. True or False: Containers include their own full operating system kernel.

<details>
<summary>Answers (click to expand)</summary>

1. VMs have a full guest OS each; containers share the host OS kernel and only isolate the application layer.
2. No OS boot process needed — containers just start a process using the already-running host kernel.
3. Image = blueprint/class; Container = running instance/object made from that blueprint.
4. Docker Client, Docker Daemon, Docker Registry.
5. `-d` = run in background; `-p 8080:80` = map host port 8080 to container port 80.
6. `docker ps` = running only; `docker ps -a` = all (running + stopped).
7. False — containers share the host's OS kernel; they do NOT have their own kernel.

</details>

---

## 12. Summary (what you should be able to teach after today)

By the end of Day 1, you should be able to explain, from scratch, to someone who's never heard of Docker:
- Why containers exist (the "works on my machine" problem)
- How VMs solved isolation, but at a heavy cost
- How containers solve the same problem while staying lightweight
- What Docker is and its 3 core architectural pieces
- The Image vs Container relationship
- How to run, view, stop, and remove a container

**Coming up on Day 2:** Deeper Docker architecture, and the core CLI commands you'll use every day (`exec`, `logs`, `inspect`, and more) — the toolkit for actually working with containers day-to-day.
